---
layout: post
title: When to optimise or not
date: '2022-03-08T17:32:00.000+00:00'
author: Luke Briner
tags: 
  - Engineering
  - Software
---

## The first rule of optimisation is "don't"
Or so you might have heard. But is that actually true?

Most arguments I find are often about two slightly different things so it is important to define what we mean by the optimisation referred to by the rule. In this case, we are talking about, "changing existing code to make it perform better". The reason the rule is "don't" is because many times, things that are assumed to perform badly either aren't or are not bad to the degree that it is worth risking the change to code to save a few cycles. All of us with any experience know that we can *still* make changes to "simple" code and break it. "Oh yeah, I forgot to trim the string", "Oh yeah I forgot to update the assembly redirects", "Oh yeah, I forgot to test that specific part of it". These are all definitely not mistakes I have made recently!

Why am I saying this? Well I think there is a type of optimisation that should absolutely be carried out and that is the optimisation of a design or architecture. If I optimise by choosing the right architecture, then that should not be covered by the rule above, otherwise it would be nonsense to suggest doing something sub-optimal. An example of this? Should you choose a List or a Dictionary/HashSet for your collection? In many cases, the trade-offs are well-known and there probably is a most correct choice. Making the right choice up-front is not optimisation, it is just ability to do something right.

There is still a danger though that even up-front, you can move from a functional implementation to a pre-optimised one. My favourite example is memory caching. Although solid-state disks and database connections are surprisingly fast, there can be a mindset of "cache-first" for all designs. If lots of requests are querying the country list, why not just cache it? I'll tell you why, because you introduce complexity, a trade-off of effectiveness vs freshness and sometimes, ironically, you still call a database method which negates everything you implemented. You might be surprised how quickly a database query can take place, so don't start adding caching until you know that these repeated calls are the performance problem (or an equal part of it).

## But it's "tech debt"
This is a really overused term and mostly doesn't mean what people think. Tech Debt does NOT mean any technical change that does not have a visible result like adding/removing caching or moving your email sending out to a microservice. They are architectural changes that might or might not be tech debt. Tech Debt is called that because it is like financial debt. You exchange a short-term gain for a long-term cost. That long-term cost could be performance but it could also be the amount of maintenance required and it might or might not rise over time. Either way, you can only pay back the debt by implementing what arguably should have been done in the first place or you live with it long enough that you eventually discard the application for a new one instead.

Does this happen? It's happen in several companies I have worked in, over a relatively short period of 5-10 years, an old application with lots of tech debt (intended or not) has simply got to the state that it is so out-of-date, that it gets rewritten either as a replacement for new customers to use or alongside the old app to allow migration.

## My rule of optimisation
My rule is simple, "Don't do it unless you know how to measure it". I suspect that many developers have not invested the time to properly measure application performance. How much overhead is the network causing? Or the database itself? Or external services? How much quicker is it to get something from cache instead of the DB in the overall scheme? How much do the limits of the browser (like max connections) impact the application? I will admit that most of the time, I cannot tell to any super-accurate degree where performance problems lie and I only usually deal with them when they obviously become a problem.

So, in conclusion:
1) Don't
2) Don't unless you know how to measure it