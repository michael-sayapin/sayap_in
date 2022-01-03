---
title: "On Rewrites"
date: 2022-01-03T14:38:57+08:00
tldr: Much has been written about software rewrites. I want to share two stories from my personal software career, one of a successful, and one of a somewhat-successful-but-terribly-wasteful rewrite.
---

# On Rewrites

Much — https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/

Has — https://steveblank.com/2011/01/25/startup-suicide-%E2%80%93-rewriting-the-code/

Been — https://levelup.gitconnected.com/why-rewriting-applications-from-scratch-is-almost-always-a-bad-idea-5402d1715006

Written — https://index.medium.com/the-story-of-every-software-rewrite-b2e83c3411d8

About — https://www.forbes.com/sites/forbestechcouncil/2021/07/29/why-you-should-refactor---not-rewrite---your-code/

Rewrites — https://gitential.com/to-rewrite-or-refactor-shakespeare-for-software-engineers/

I will not repeat those points. I agree with them. Here I want to share two stories from my personal software career, one of a successful, and one of a somewhat-successful-but-terribly-wasteful rewrite.

## Tale of the Successful Rewrite

Back in 2007, a year fresh from the college, I was freelancing and half-assedly looking for a new job, sending my resume here and there mostly on a whim. Freelancing pay was okay, albeit the projects were not challenging. Squarespace and even WordPress was not a thing yet, and most of those mom and pops small business websites were handcrafted.

One day I got a call. A very distressed guy on the other end said: "You say on your resume your strength is solving problems fast? We need that now. Can you come to this address now. Pay is not a problem." I was taken aback but rather intrigued. It was 2 or 3pm on a working day, but I had free time, so I went.

It was a small, badly lit office, but smack in the heart of the city. I lived nearby, so it took me just a few minutes. They seemed happy.

From a barely coherent explanation, I understood that the company is a somewhat experimental e-commerce arm of a large brick-and-mortar group, and after some escalation with their development "team" (consisting of 2 people), the company refused to pay and the team stopped their website and held the source hostage.

Their boss dug the heels in and "refused to negotiate with terrorists", so the situation was unfolding fast.

From the get go I saw that the "terrorist" team made a bunch of mistakes:

- they turned off the (two) servers via `shutdown -h now` so they locked themselves out too,
- this was either badly planned or spontaneous, so their company PCs were at the office, which they just lost access to.

Within a few hours we managed to retrieve the servers and boot them in single mode (which means instant root access), change the `passwd`, remove all SSH `authorized_keys`, and ship them back to the ISP.

The PCs — which nobody except the team knew the passwords to (no sysadmins/ops were in the company at the time) — also readily gave up their HDDs which turned out not to be encrypted. Yay. I had the source.

The whole conundrum landed me the job as the Team Lead (with no team yet) on the spot.

However, when I saw the "source" I was truly horrified. The website was basically several parts: catalog, order/billing, and backoffice. The "source" had three files too: `catalog1.php`, `order.php`, `_manager.php` — each one well over 10,000 LoC, and each one a sumptuous mix of PHP spaghetti, HTML, and SQL. It all talked to a MySQL DB designed in a very similar way. The folder also had some other files like `qqq.php` and `q.php` and `sdfsfsds.html`.

It was a disaster.

And I just agreed to inherit a very unique set of problems:

- the original team was fired, so I had to figure it all out alone, and fast,
- the website still had about 30-50 orders per day which had to work,
- there was no time to hire any additional people,
- the warehouse, photo studio, deliveries were supposed to continue working as well,
- on top of that, they just printed a bunch of ads in a well-known magazine!

So, after patching a few gaping SQL injections typical for PHP circa 2006 (a la `/catalog.php?item=1; DROP DATABASE db; --`), I sat down with the management and laid out my plan...

### Incremental business functions replacement

Since the Spaghetti a'la Pehepese were clearly unmanageable, they had zero comments or anything, and I had very limited time, I decided to keep everything working as-is, and replace one "function" at a time, starting with less critical pages, and eventually the whole website page by page. Then the back-office.

On a few previous freelance gigs, I tried this Shiny New Tech (typically, a red flag of any rewrite!) called Django. I made a few websites using v.0.91, and heard that a new version, v.0.96 just came out that addressed many shortcomings. It worked well for me in the gigs, and the motto of "The web framework for perfectionists with deadlines" helped me convince the management. And oh boy, I had deadlines.

### Decouple and keep the database

To convince the old PHP code to work together with new Django code, I kept the database running. It was badly designed, but I felt that trying to wrestle it into some kind of "normal form" would break more things than it would fix.

So I used Django's `inspectdb` and generated a bunch of models. Many did not have any primary keys or any indexes, so I added that, but overall I left the DB in the same state.

After that it was fairly easy to whip up a bunch of Apache `.htaccess` files (still a thing in 2007) to "steal" some requests from PHP and serve them with Django.

### 60% time on rewrite, 40% time on creating new value

However, of course, no rewrite happens in vacuum. As I mentioned before, the company was an experiment in e-commerce by the retail chain boss, and it turned out that it was his "pet project". He had great interest in what we were doing, and, what's not great, he had many many "suggestions" on how to improve things :-)

So in a very short time I found myself in the same boat as many other "rewriters": how to keep up with the business demand while trying to migrate a legacy codebase?

No how, it's impossible. It adds a bunch of bloat, you have to do everything twice, or you have to say "no" a lot, which doesn't make you any more popular.

So we decided that we will migrate customer-facing parts first, and dedicate two days a week processing "tickets" from the boss and other stakeholders. This way we could show some progress, while actually spending more time on the rewrite. If a new feature was touching a still-legacy part, we found excuses or my project director (who was a top guy!) produced exhaustive lists of questions, so the feature would be in discussion for a while — all to buy me some time.

### High-level testing for development speed

Of course, when you're doing an incremental rewrite, you have to make sure there are no regressions. I didn't have time enough to write detailed unit tests for everything except core business logic and billing, so I took a path of least resistance.

A Selenium script placed an order on the production website every 5 minutes. Then the order was automatically deleted. If it didn't complete, I got an email warning. Simple as that, but it tested the "critical path" (i.e. the thing that really matters), was complete in a day, and saved my ass countless times.

After that, it went smoothly. In less than 3 months, "The Rewrite" was complete, and we even released a new back-office order processing page using another Shiny New Tech (ouch!) — jQuery! :-)

Back-office people were happy (the pages were infinitely faster), my project director was happy, the boss was reasonably nonplussed (he turned out to be that "I only delegate because I'm too busy, I could've done it myself much better, of course" kind of guy).

We even grew a few tens of percents during that time, to 50-60 orders per day. It was a long way to thousands of daily orders we saw in a few years, but it was a good start.

And I stayed true to Django (and allergic to PHP) ever since. It lived up to the promise of being "the web framework for perfectionists with deadlines", and we all have deadlines. If you don't have, either you're in a bubble (enjoy it while it lasts), or it's your last year in business. Rethink it.

Either way, the morale of the story is:

- ~~don't get in arguments with your developers~~,
- only rewrite if you're absolutely cornered,
- if you choose to rewrite, do it incrementally,
- do not rewrite "in isolation", put rewritten parts out ASAP,
- have a suite of integration tests, but don't spend too much time on it,
- dedicate 40/60 or 50/50 of time to porting and new features, the market doesn't care about your rewrite.

## Tale of the somewhat-successful-but-terribly-wasteful rewrite

This story is much less dramatic than the previous one, but nevertheless a good example of what you're getting into, as a company, when deciding to rewrite.

The backdrop of the story is a small startup working on an IoT solution to optimize enterprise energy performance. It has a lot of moving parts: hardware packed with sensors, firmware that runs on this hardware, backend in the cloud that deals with all the data, and frontend that displays it nicely to the customers.

When I joined as a contractor, it was all pretty bare. Backend was a single Python file keeping all data in memory. Firmware was clearly smacked together from examples (complete with example comments). A few customers. Decent custom hardware and a pretty good frontend though.

Technically, writing a new backend for them was a "rewrite", since the company had a Python file. But it was one of those situations where the project was small enough and with a lot of kinks to justify doing it from scratch. So in less than a month the "first contact" between hardware and the new backend was made.

The firmware evolved separately, and its code quality was not great. It was written by a very smart developer, but with barely any experience in C, it had all kinds of memory leaks, unexpected shutdowns, core dumps, and other stuff that rids the embedded programming world.

Since the original developer was remote and just got another job somewhere, I was left to patch and evolve this firmware to keep up with the business. In a few months it was reasonably stable, but since I'm also not a C programmer by any stretch, the code quality still was not great.

During that time, we designed/evolved a new, Protobuf-based protocol to replace the old JSON-based protocol, cutting traffic 10x, and getting rid of all the magic strings and informal expectations on both sides. The replacement went well, with the same approach as above: we kept the logic very similar, and for a while supported both, then slowly did a bunch of over-the-air upgrades to the devices so eventually they all spoke Protobuf. JSON was then removed.

My contract time was ending, I had another big gig coming up, so I left the company, only paying attention and supporting them a few days per week, to keep an eye on the new backend and its stability.

However, of course, the business continues evolving, and soon they found themselves needing more features in the firmware. But nobody in the team knew C really well. They engaged the ex-developer who agreed to dedicate a few hours per week.

Then they made a mistake.

Since the ex-developer was practicing C on his new job, he learned a lot, and his old code (with my clumsy fixes) was not up to his snuff. So he proposed to rewrite it from scratch, with a classic software estimate of "a few months". They called it "Ferrari firmware" since, of course, after rewrite it was supposed to be sleek, shiny, and fast.

The founder somehow agreed to that.

Feeling the invigorating rush of Rewriting, they decided since they do it, why not also rewrite the Protobuf Protocol?

Great, two rewrites at once, in a team of 3. What can go wrong.

Fast forward "a few months", of course, the project status is basically on the back burner. The C developer got busy with his job, nobody could really help, so everything stalled.

Here's your chance to cut your losses! They decided to persevere.

In the meantime, the business was selling hardware with the old, patched firmware, and since I had some extra time again, I agreed to help in patching it further and add new features. Yes, to the old firmware that was being rewritten, because that's what was sellable. So the goalposts started shifting forward and forward, and nobody could catch up to them.

Long story short, this dragged for more than a year, ate 20+ developer-months, but was eventually finished.

However, it was finished together with the "new" Protobuf protocol, which the backend did not speak, since it was not an incremental improvement, but a complete shift.

We spent another 3+ developer-months trying to wrestle the backend to speak both "old" and "new" protocols at once, and this work is still ongoing, with my conservative estimate that another 6-9 months needed to finalize it.

Was it worth it?

Yes, the new firmware code quality is miles better than the old. Could they incrementally improve the old code using those 20+ developer-months (or, probably, much less)? The two firmwares still didn't quite reach feature parity, so this saga is still ongoing.

Ironically, the new protocol is better organized, but more than twice the traffic on the wire, with lots of fixes being added weekly. Could they do a proper evolution using Protobuf's excellent versioning capabilities?

Remember the Gall's law:

> A complex system that works is invariably found to have evolved from a simple system that worked. A complex system designed from scratch never works and cannot be made to work. You have to start over, beginning with a working simple system.

The morale of the story is:

- when a developer says "few months", ask for a working prototype in 4 weeks, and keep track of time from there,
- cut your losses when it's clear that the project will take 10x time,
- it's okay to throw away / rewrite a small MVP if you still have an original team, with all the learnings from this MVP — don't do it if you don't!
