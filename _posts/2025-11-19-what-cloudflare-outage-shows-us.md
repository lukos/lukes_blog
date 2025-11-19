---
layout: post
title: What the Cloudflare Outage of November 18th 2025 shows us
date: '2025-11-19T10:14:01.000+00:00'
author: Luke Briner
---

## What Happened at Cloudflare on Novemeber 18th?
Cloudflare, an international network/cloud-based web application proxy/firewall/DDoS protector that protects many 1000s of large websites globally became unavailable for over 3 hours, while the long-tail of recovery took another 3!

This was Cloudflare's longest outage since 2019. Since Cloudflare protects many large and famous websites, all of these were unavailable for most or all of the window from 11:20 to roughly 17:10, this was [national news](https://www.bbc.co.uk/news/articles/c629pny4gl7o) in most countries.

What actually happened? As usual, a chain of events.

1. A change was made (deliberately) to a set of database permissions
1. The effect of this, unknowingly, was to increase the size of a generated configuration file by roughly double
1. This file was pushed out over a number of minutes to the systems that use the file
1. These systems are deliberately memory constrained and the file was too large to open so they would crash since that scenario was not expected/handled cleanly
1. It was not clear what was causing the problem partly due to a) recent DDoS attacks meaning this might be another one b) The staggered deployment meant that some systems were OK and others were not c) The Cloudflare status page, which was independent of all of this was having unrelated issues making it look even more like a DDoS attack, which led to the large time to diagnose
1. Once the problem was identified, an older/smaller file was manually pushed into the deployment queue and a number of systems had to be restarted
1. As with all large systems, backpressure builds up from batch processes that are sitting retrying to access things so even after the system is up again, it then has a deluge of work to do, whic often reduces performance for a few hours as things catch up.

To make things worse, customers would not usually be able to login to the Cloudflare Dashboard to make any changes e.g. to DNS to workaround the problem, since the Turnstile "captcha" on the login page was taken out by the breakage! Ouch.

Most of us sat there weeping, hoping it would all be fixed soon.

Cloudflare, to their credit, published a full and transparent post-mortem on the incident [here](https://blog.cloudflare.com/18-november-2025-outage/)

## What *didn't* go wrong?
It's very easy for those of us who don't manage critical infrastructure of this size and complexity to come out with, "they obviously should have done....", which is very patronising to a group of very talented staff. I read an article once about them finding a bug in a CPU! who has ever done that?

Anyway, let us knock down a few of the trite accusations levelled at CLoudFlare:

1. How does a company with this much money/staff/skill make an error like this? Answer: None of those things prevent mistakes/bugs/issues, they just reduce their chance. The complexity of their system already magnifies their risks way beyond what most of us are managing.
1. My system has never gone down for this long! Answer: Congratulations. If your system is as complex as Cloudflare's go and get a job there, but I suspect it isn't. Here at SmartSurvey, we have traditionally had amazing uptime but guess what? We are orders of magnitude less complex than Cloudflare.
1. Why didn't they update the status page sooner? Answer: I don't know how long it took, but I don't remember it being very long. Also, there were other issues with the status page that wouldn't have helped and those people who usually update it might well have been fighting other fires that started. Another problem with large businesses is you get so many reports of "this is not working", it does actually take time to confirm the problem before you update the status page and the tickets themselves do not get looked at instantly.
1. How can a system as important as this go down for as long as this? Answer: This is a non-sequitur. A system being important doesn't, in itself, make that system more reliable or make it any easier to make it reliable. The truth is that 99.9% of the time, Cloudflare provides very valuable functionality and services that didn't really exist before the company was created (a few alternatives are available now but not to the same degree). If you could choose 3 hours downtime per year with DDoS protection or no DDoS protection, you will likely still accept the risk of downtime.
1. Why didn't they have a rollback plan? Answer: They probably did but for that to work, the failure needs to be expected so you can watch out for it and if there is no obvious link between the update and the failure (remembering that Cloudflare probably update loads of things multiple times per day), then you aren't going to know to rollback, especially if rolling back also undoes other critical updates and you don't know what update broke it!

## What *did* go wrong?
When taken in its entirety, it sounds bad, "Code didn't cope properly with large files" but, like many disasters, when broken down, each element is not that bad in its own right.

1. Deployed a database permissions change. These have to be done sometime and on-paper, the change was non-breaking. The issue wasn't that the change was wrong but that a previous SQL query assumed something that was no longer true after the change. How would you spot that in testing? Not very easily. Even if you did run it in testing, nothing broke at this point, the generated file, if it is part of the test suite would have been larger but still valid so not really a screw up here.
1. Ah but whoever wrote that original query should not have made assumptions! GOod luck with that, we all make assumptions every day. We cannot predict the future so we have to work with what we know now. The truth is that neither side of this did anything majorly wrong but the result was wrong. It was, however, only wrong because of another assumption...
1. The bot detection agents are memory constrained for performance reasons. I assume this means we don't let things randomnly allocate whatever memory they want because that could cause performance issues that would be hard to debug. This meant that when the larger file was opened in code, there was not enough RAM and Rust threw a panic. Or rather Rust returned a result which the Developer did a panic on. This is slightly worse in my opinion but there was also possibly a disconnect between the Developer who had to code "Open the file to read the contents" and someone else who decide that "the file is only 40MB, let's set a RAM limit of 200MB", which is plenty, although arguably, instead of a Panic, the developer could have logged a specific error, which could have been aggregated by whatever Cloudflare monitor their system with so that as soon as there were problems, they would have seen "Error in Bot services: Out of memory opening feature file", which might have led to a quicker conclusion.

## What did we learn?
We learned that we don't have the tooling to reliable link all of these things together. We don't have a way to markup a SQL query with "this assumes that only the default database is accessible" in a way that means if someone makes a permissions change, somehow, the assumption is surfaced to allow someone to action it. It doesn't allow someone to annotate that, "Feature files are never supposed to be over 40MB so it is OK to panic if we run out of memory" so that, at runtime, a large feature file triggers something that highlights the danger (a log might have solved that to be fair)

Don't assume something is a DDoS until you have verified that. This made some data get interpreted correctly and make it take longer to resolve.

Don't make your control panel unavailable by using Turnstile on your main infrastructure ;-)

For us as customers, consider what a 2/4/8 hour outage of Cloudflare means. Can you work around it even at reduced functionality? What if you cannot access Cloudflare to change settings over?

We all need to have good responsiveness to updating status pages, which MUST be hosted on separate infrastructure to our own.

Don't use a backup DNS provider that uses Cloudflare! Yes, there are some.
