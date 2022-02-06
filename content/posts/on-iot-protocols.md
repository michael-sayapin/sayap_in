---
title: "On Iot Protocols"
date: 2022-01-04T15:14:04+08:00
tags:
- iot
description: When designing an IoT device, one of the first questions you need to answer is how are you going to receive data from the device. There is no single industry standard, and many protocols have their unique pros and cons. I will try to summarize the current (2021) most popular choices and their advantages and disadvantages in specific scenarios.
---

When designing an IoT device, one of the first questions you need to answer is how are you going to exchange data with the device. There is no single industry standard, and many protocols have their unique pros and cons. I will try to summarize the current (2021) most popular choices and their advantages and disadvantages in specific scenarios.

## Raw TCP

Raw TCP socket is the simplest way of reliable communication. The device connects to a specific server, typically, by IP address and a specific port number. On the device side, the connection state is tracked, and most C-based implementations will use efficient `select/poll/epoll`-family of polling calls to notify if there is incoming data.

After a connection is established, you can write and receive any amount of data from the socket. Underlying TCP stack will take care of IP formatting, fragmentation, acknowledges, and reassembly.

In the ideal world, TCP socket is the simplest and stablest link.

However, we do not live in the ideal world. Sometimes network can go down, or the server can malfunction and stop reading the data. In this case, TCP socket can stay "connected", but actually stop working.

Usually, this is solved by some kind of keepalive or heartbeat mechanics, which involves implementation on both client and server, whereby one of them is sending "pings" and waits for some time, if "pong" is not received, the connection is closed and re-established.

TCP socket can be wrapped in Transport Layer Security (TLS) stack to transparently encrypt data (if the server supports that). Most "larger" (>100KB RAM) embedded platforms will also contain some form of TLS, usually, via MbedTLS or WolfSSL libraries.

Advantages:

- simplicity and availability on a wide array of platforms
- message delivery and order guarantees
- message size is not limited by the protocol
- largely effortless TLS security if you have 24KB+ RAM to spare

Disadvantages:

- resource-hungry (on embedded) due to constant socket polling
- transitional network states and disconnects are hard to handle properly
- server-side scalability for multiple concurrent connections can be an issue (aka "C10k problem")

## Raw UDP

UDP is a connectionless protocol operating on short messages called "datagrams". Strip all the connection logic from TCP and you will end up with UDP.

Sending a UDP datagram is like sending a letter to some address. You drop it at the post office, and you can go on with your life. You only know if the addressee received it if you explicitly call and ask.

Since there is no connection establishment, UDP protocol is very lightweight and well-suited for constrained devices, and in scenarios where lost or delivered out-of-order messages are okay.

Also, typically, UDP is a one-way street, there is no way to send a UDP "command" to your device unless it's running a UDP server on a routable IP address (very unlikely).

However, there **might** be a short time window for the server to reply to your UDP message, if necessary. This is not guaranteed and handled by the network equipment via Stateful Packet Inspection (SPI). When the device sends a UDP message, the router(s) on the way will keep the "street" open both ways for a short while, typically around 20-40 seconds. Again, this is not guaranteed.

Also, UDP packets are supposed to be short bursts, so usually, network equipment on the route will fragment (i.e. split) packets over, typically, 1500-ish bytes (network MTU). Your application needs to handle reassembly if you need larger packets (and it is a very hairy topic).

Security-wise UDP is also very complicated since it doesn't have any connection state for TLS handshake and stream encryption. Some efforts have been made with DTLS, but for embedded platforms, it is rarely supported.

It's worth noting that a lion's share of IoT cybersecurity issues and attacks were possible due to UDP usage.

Advantages:

- very resource-friendly
- simple to implement if no guarantees or ordered delivery is necessary

Disadvantages:

- impossible to issue an ad hoc message to a device, only a response
- no guarantees of delivery or order of delivery
- fragmentation over 1500 bytes
- hard to secure

NB: **every protocol based on UDP** suffers from the same issue — without a connection, there is no way to reach back to the device at an arbitrary time, e.g. to issue a command, or initiate a firmware update. Servers must wait for the device to communicate to them and then reply over unstable and non-guaranteed SPI links.

## COAP

To use UDP's lightweight advantages and overcome some of the application- and transport-level issues, the IETF group designed CoAP (Constrained Application Protocol).

Most importantly, CoAP protocol handles request/response, fragmentation, and ping/pong patterns over UDP links, so you do not have to implement them. It is modeled after Web HTTP RESTful protocols and is conceptually based on the same idea of "verbs": `GET`, `POST`, `PUT`, `DELETE`, `FETCH`, `PATCH`.

By default, the message, like its underlying UDP protocol, is not guaranteed to be delivered. To combat that, CoAP defines an optional **confirmable** flag, which requires a response to the message. If no response is received, the message is resent. Every message has a unique ID, so both peers can filter out duplicate resent messages.

If the device is sending larger amounts of data, it is possible to utilize `coap_keepalive` mechanics which is just an empty confirmable message sent every few seconds, to keep the SPI cache along the route alive (the technique known as "UDP hole-punching" — not every network administrator will be happy about this).

CoAP fundamentally suffers from the same issues as its underlying transport, UDP. It's hard to guarantee response delivery since UDP was not designed for that (with a notable exception of DNS and ICMP). Message sizes are also limited by either link-layer size (64K) or IP packet size (typically 1024 or 1280 bytes).

It's worth noting that IETF also designed RFC 8323 "CoAP (Constrained Application Protocol) over TCP, TLS, and WebSockets", but it has yet to see much adoption.

Advantages:

- reasonably resource-friendly
- easy to understand semantics similar to HTTP REST
- typically available in AT-based firmware of cellular IoT modules

Disadvantages:

- same as raw UDP
- UDP hole-punching might make you unpopular at SysOp parties

## MQTT

MQTT stands for Message Queue Telemetry Protocol, however, the "queue" there is for historical reasons. This protocol is based on TCP connections and handles message transport, keepalive, and connection state management.

MQTT is a protocol that needs a "broker" server, though, from a purely technical point of view, a broker is just a TCP socket server conforming to certain requirements.

In MQTT there is a clear definition between the clients and the broker. Clients can send messages to certain "topics" and subscribe to other topics. Thus MQTT can efficiently decouple producers (devices that send e.g. sensing data) and consumers (backend scripts that process and save or react to this data). Topics are typically `/`-separated patterns, and support basic globbing using `+` ("anything except /") and `#` ("anything at all") characters.

Thus one of the most interesting features of MQTT is its publish/subscribe pattern. If you need to send messages to 10,000 devices, with raw TCP or UDP, or COAP you need to send one by one. With MQTT you make sure the devices subscribe to a specific group topic and send one message there. It will be delivered to all your devices. This also means you can send a message from device A to device(s) B, and they do not need to share the network.

It is a duplex protocol as well, you can send messages to the devices at any time. MQTT connections keep their health status by using ping-pong keepalive. When connecting, the *client* is declaring that it will send pings with a specific interval (e.g. every 60 seconds). When the server did not receive a ping in 1.5x this interval (i.e. 90 seconds), it will close the connection and declare the client dead. During connection, the client can also declare a special "last will" message, which will be published by the broker at this moment, so you can process these disconnection events.

MQTT defines several levels of message delivery guarantees (aka "QoS"): 0 or at most once, 1 or at least once, and 2 or exactly once. QoS 0 is "fire and forget", if the connection was bad or the recipient is overloaded, the message is lost. QoS 1 is acknowledged by the broker, and if not acknowledged within a certain time frame, it is resent (similar to CoAP's confirmable messages). QoS 2 employs a sophisticated 3-way acknowledge (acknowledge the acknowledge), and is typically not used with constrained or embedded devices.

Clients can connect with a boolean "clean session" flag. If it is not set, any QoS 1 or 2 messages that might have accumulated for this client and were not acknowledged are resent. If it's set, all the messages for this client are discarded.

Publish/subscribe nature of MQTT is also its weakness: if your application needs some kind of request/response pattern (e.g. server: "turn on LED"... client: "OK! LED turned on!"), with MQTT you need to handle this in the application layer.

Since MQTT works over TCP, it can be secured with TLS. It also provides username+password or certificate authentication mechanism, and some brokers have means to authorize topics via some kind of ACL (otherwise any device can subscribe to `#`).

There's also a plethora of brokers to choose from, which is a great practical strength of MQTT. To avoid a single point of failure (SPoF) situation, it is possible to deploy clustered brokers like VerneMQ, HiveMQ, or RabbitMQ in MQTT mode. Apache Kafka also has a separate proprietary MQTT Proxy product that can help with ingesting IoT messages directly into Kafka.

Some brokers also provide an ability to transport MQTT over WebSocket `ws://` or `wss://` endpoints, which means it can be proxied via Cloudflare, ingressed by Kubernetes Ingress, load-balanced, and does not need any exotic ports in the firewall. SysOps and Security departments adore this! And yes, you can talk to WebSocket MQTT from JavaScript.

For Cloud-loving teams, it's possible to use MQTT via AWS IoT, Azure, GCP, and many other cloud providers.

Finally, most brokers can themselves be the clients of other brokers (what is called "bridging") and build sophisticated message relay schemes. In the IoT context this is typically used to isolate the "edge" and the "cloud", deploying a minimal broker close to the devices for faster or isolated connectivity and bridging these messages upstream to the main broker.

It's also worth mentioning that most IoT and embedded devices use MQTT v.3 standards, but as MQTT v.5 came out in 2019, it's starting to get some library support. Some of the highlights include more flexible authentication, client load balancing, request-response pattern support, rate limiting, flexible topic rewrite, and more. However, it's still quite early to use it in IoT.

Advantages:

- one-to-many and many-to-one messaging patterns
- reliable delivery semantics
- low protocol overhead
- typically available in AT-based firmware of cellular IoT modules
- upcoming MQTT v.5 features

Disadvantages:

- as raw TCP, this protocol can be resource-hungry
- request/response pattern can get complicated
- needs separate broker software

## MQTT-SN

MQTT-SN stands for "MQTT for Sensor Networks", and it's a continuation of the idea of "edge isolation" mentioned above. If you have a local MQTT sub-broker at the edge, why not strip it down and make the protocol more suitable for very constrained devices?

SN does that. In a nutshell, it's a re-implementation of MQTT that works over UDP, XBee, or, technically, any other medium (e.g. RS485 or Bluetooth or LoRa). Since it works on the edge, UDP is typically more reliable since it doesn't need to traverse the Internet with its many NATs and opinionated network equipment.

MQTT-SN allows publishing and subscribing to the topics by numeric ID. The list of IDs is a developer's responsibility. This removes quite a few bytes from the payload (every MQTT message contains the topic as a string, so e.g. "sensor/temperature/room1", 24 bytes, is sent in every single message).

Another one is, of course, connection virtualization. Operating over UDP means that the connection semantics are much simpler, and there's no lengthy exchange at the first stage, and on the device side, you do not need to keep the connection open.

MQTT-SN does not have a broker, instead, it has a Gateway. Which is kinda broker, but limited and working over UDP or other links. Typically, these gateways have settings to forward messages to a "real" MQTT broker upstream.

While this all sounds like a good idea, currently MQTT-SN development is stalled, there are not many good quality libraries, and I cannot recommend using this protocol. But it's a good one to keep an eye for!

Advantages:

- all of MQTT's goodness with reduced traffic and memory footprint
- ability to work over heterogenous links

Disadvantages:

- not actively developed or maintained
- packet length limitations and concerns driven by UDP

## LwM2M

Lightweight Machine to Machine protocol is a more opinionated IoT protocol that includes, beyond simple messaging, specifications for device management, provisioning, lifecycle, and configuration.

Original LwM2M v.1.0 circa 2017 used CoAP over UDP (or, yikes, SMS) as transport, version v.1.1 added TCP with optional TLS security, and the freshly baked v.1.2 (out in Nov 2020) supports HTTP and MQTT as transport.

This protocol operates on a structured representation of devices:

    Object -> Object instance -> Resource -> Resource instance

Object is a functionality of a general device. Device implements them as Object Instances (in terms of a connecting peer). Objects contain Resources. Each Resource is a specific characteristic of the device, that can be Read from (ad hoc or Observed), Written into, or Executed. Resource Instances allow multiple values in a single Resource, e.g. arrays of values.

The standard provides an eye-watering table of available pre-registered Objects and Resources at https://technical.openmobilealliance.org/OMNA/LwM2M/LwM2MRegistry.html — on top of that, companies register thousands of their Object identifiers. Please don't ask me why 501 is a `LTE-MTC Band Config` and 502 is a `CO Detector`.

Maybe 154 pages of "Core" standard at https://www.openmobilealliance.org/release/LightweightM2M/V1_2-20201110-A/OMA-TS-LightweightM2M_Core-V1_2-20201110-A.pdf will explain it better?

> As an illustration, the following example using the discover operation on the Server Object, exposes a configuration in which the Server Object (ID:1) has 2 Instances (ID:0, ID:1): the Optional Resources ID:2, ID:4 are only instantiated in the Instance 1 of the Object, while the Optional Resources ID:3 and ID:5 are not instantiated in either of the Server Object Instances. In Server Object ID:1, the Resources 0,1, and 6,7,8 are Mandatory Resources. According to the DISCOVER /1 request, the following payload is returned: `</1/0/0>,</1/0/1>,</1/0/6>,</1/0/7>, </1/0/8>, </1/1/0>,</1/1/1>,</1/1/2>,</1/1/4>,</1/1/6>,</1/1/7>, </1/1/8>`

Nah.

Data can travel in LwM2M in several formats: plain text, binary, CBOR (tightly packed binary JSON), TLV (tag-length-value, somewhat similar to Protobuf), JSON, and some special forms of JSON. This partly depends on the Object type (e.g. "manufacturer" Object is plain text), partly down to the developer's choice.

If all of this sounds overly complicated, it's because it is. As an IoT protocol, LwM2M tries to provide a framework for **everything** — transport, exchange format, bootstrapping, discovery, reading and writing, event streams, some kind of RPC, access control, and so on, and so on.

It's a very hairy job to be able to do all these things equally well, and I have a strong feeling that choosing LwM2M has a lot of non-reversible impact on all of the other parts of the IoT solution (i.e. firmware, backend, even hardware). It sounds like something that would be close to impossible to opt-out of, if necessary.

It might be a good choice for some kind of industrial gateways or complicated PLC equipment with hundreds of data points, produced by a big vendor like Eaton or Murata. For IoT startups, this sounds like a swamp to get stuck in. YMMV.

Advantages:

- tries to do everything

Disadvantages:

- tries to do everything

## AMQP

AMQP or Advanced Message Queuing Protocol is a messaging protocol from the same group that develops MQTT standards, OASIS.

Compared to MQTT, which tries to be as lightweight as possible (there are complete MQTT broker implementations in ~1000 LoC), AMQP brings much more features and patterns, defining not only the message layer (i.e. packet format), but also application *behavior*.

Clients can publish messages and subscribe to "queues" (very similar to topics in MQTT), but in AMQP a Queue has more features.

The main difference is that by default AMQP queues are durable. In MQTT (v.3), if nobody is subscribed to a specific topic (and there are no pending subscribed clients with "clean session = false"), messages sent to it are discarded by the broker. In AMQP, the messages are persisted and will be delivered later. Most brokers also expose per-queue or per-message TTL and / or maximum size, and often rate limits per client as well.

AMQP queues can distribute messages evenly between subscribed clients, while MQTT (v.3) topics can only fanout (so MQTT is not suitable for Distributed Worker pattern where consumers grab new a message to process as soon as they finish working on the previous one).

Consumers in AMQP can flexibly acknowledge message processing, i.e. do the "work" and then signal the broker to delete the message, or error out and send a negative acknowledge (NACK), so the message is back to the queue and another consumer can retry it. The message can also be automatically routed to a "dead letter queue", which allows for very flexible erroneous message processing.

However, of course, all of it comes at a cost: AMQP brokers are more resource-intensive, and clients are much more complicated. It is very rare to use AMQP for embedded and IoT devices. This is a protocol more suited for server-to-server communication, IPC, or event-driven architectures.

One notable exception is Azure uAMQP lib at https://github.com/Azure/azure-uamqp-c — which is currently much of a work in progress, but worth keeping an eye on!

Advantages:

- flexible routing semantics
- message persistence
- delivery guarantees and ACK/NACK on protocol level

Disadvantages:

- complexity
- resource-hungry for embedded use

## HTTP

Everybody knows HTTP, so this might seem an easy choice for a simple IoT application. What could be easier? `POST` readings to your API, poll `GET` for new commands, and everything should "just work" ⓒ

However, HTTP as an IoT protocol has numerous disadvantages. Take a typical HTTP request and response as an example:

    $ curl -v -k -H 'Authorization: Basic T0Ps3c3t' https://iot.example.com/api/v1/device/DEADBEEFCAFE/commands/

    GET /api/v1/device/DEADBEEFCAFE/commands/ HTTP/1.1
    Host: iot.example.com
    User-Agent: curl/7.74.0
    Accept: */*
    Authorization: Basic T0Ps3c3t

    HTTP/1.1 200 OK
    Access-Control-Allow-Credentials: true
    Access-Control-Allow-Headers: DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization
    Access-Control-Allow-Methods: GET, PUT, POST, DELETE, PATCH, OPTIONS
    Access-Control-Allow-Origin: *
    Allow: GET, POST, HEAD, OPTIONS
    Content-Language: en
    Content-Length: 2
    Content-Type: application/json
    Date: Thu, 06 Jan 2022 13:26:11 GMT
    Referrer-Policy: same-origin
    Strict-Transport-Security: max-age=15724800; includeSubDomains
    Vary: Accept, Accept-Language, Cookie, Origin
    X-Content-Type-Options: nosniff
    X-Custom-Header: 7g4b4isrLuS0IfpdXJooa6V1cA2AvkOQwJza0wRIvPKWTbD0bpnPzZZBb==

    []

We just basically transferred >900 bytes (+ SSL overhead of ~30%) to receive TWO: `[]`, for a total overhead of a whopping 50000+%. If you have a stable connection, not too many devices, and do not care about bandwidth, this might be a reasonable choice for its simplicity. For IoT projects that are aimed at scalability or are using constrained and unstable traffic, HTTP is definitely an overkill.

Advantages:

- ease of development
- no custom server software

Disadvantages:

- bloated text-based protocol with limited control over response size
- need to constantly poll the server for commands

## WebSocket

After HTTP, the reasonable next step might be WebSocket, which is basically a raw TCP stream established over a persistent HTTP channel.

Messages over WebSocket can be binary, and it's easy to send and receive arbitrary data. Communication typically happens over HTTP ports 80 or 443, and TLS can be handled by the standard means, e.g. `cert-manager` for Kubernetes, or even proxied with Cloudflare or analogs for a hands-free SSL certificate.

However, all the same benefits can be obtained by using MQTT over WebSocket, which will also add mechanics to implement publish/subscribe patterns, delivery guarantees, authentication, and more. So there's no real niche for using pure WebSockets in IoT, unless you just want to stream some data, e.g. audio or video, from a device directly into a web browser.

Advantages:

- binary TCP with low overhead
- ability to use existing tools for security

Disadvantages:

- less feature-rich than MQTT with basically the same overhead

## gRPC

Another possible protocol that works over HTTP is gRPC, Google's Protobuf-based Remote Procedure Call (RPC) protocol. Reportedly, the whole Google uses gRPC to communicate between all, probably thousands, of its microservices. Later it has been adopted as one of CNCF (Cloud Native Computing Foundation) projects. This protocol automatically handles argument serialization, retries, streaming, and authentication. Obviously, it's also the epitome of request/response pattern.

gRPC standard includes implementations for "unary" (request/response), "server streaming", "client streaming", and "bidirectional streaming" procedure calls. All of this happens over HTTP/2 streams and/or WebSockets.

In theory, this all sounds very interesting, but currently, gRPC support is very limited on embedded IoT platforms like AVR, ESP. Only higher-level Linux-compatible platforms like Raspberry Pi can run gRPC, so it's not a very optimal choice for IoT projects working with constrained hardware.

Advantages:

- clear mapping of calls to procedures on client and server
- flexible and secure authentication
- automatic failover and retry mechanics

Disadvantages:

- limited support for popular IoT platforms
- HTTP/2 can be resource hungry even on larger platforms

---

To summarize, MQTT is the usually the simplest choice if there are no sophisticated integration requirements. Brokers are available from all major cloud platforms, and libraries are there for most embedded frameworks.

Hope this information was helpful.

As always, if you want me to help out in choosing a protocol, or have any questions, do not hesitate to reach out at `michael [at] sayap.in`.
