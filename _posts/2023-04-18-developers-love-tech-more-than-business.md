---
layout: post
title: Developers love tech more than business
date: '2023-03-18T11:10:00.000+01:00'
author: Luke Briner
tags: 
  - Strategy
  - Culture
---

## Who would have thought?
The title might sound like a case of stating the obvious but I am not just saying that Coders gonna Code but if you look at the Technology industry in its entirety, these are the following sorts of problems:

* Inconsistent UI/UX
* Apps which are fragile
* Using async/online-only apps for desktop
* No consistent training/qualifications for "Developers"
* Front-end apps where they are not warranted
* Superstar culture
* High salary expectations
* A lot of form over function
* All kinds of application performance issues

Some of these might not seem obvious problems but let us take just one of these: Front-end apps where they are not warranted. What are front-end apps good for? There are only 2 reasons I think we should ever consider them:

1. You have enormous numbers of users and want to push work to the browser
2. You are doing something really extraordinary that is too hard to do with simple Javascript and HTML

Other reasons like "Your devs can do front-end and backend" is a non-reason as far as I am concerned. I have literally never seen a case where this is an issue. Back-end devs can mostly do a reasonable amount of front-end code and front-enders almost never want to write anything backend.

So what? What are the downsides of using a front-end app?

1. Almost always at least an order of magnitude more complex. At minimum, we are usually talking about front-end code + an API endpoint. We are also talking about session management, maybe JWTs, duplication of authorisation in two places.
2. There is additional time involved for dev e.g. the additional latency of having to do both FE and BE pieces (possibly with builds) before the work is completed and then the significant time that something like yarn or npm take to run webpack. In our .Net app, the npm build (for probably 10% of our pages) takes 5 times longer than the .Net build.
3. FE apps are supposed to move work to the browser but in most cases, a single FE page is making 10s of calls to APIs, each of which is almost a BE app in its own right. Most people save nothing by avoiding session since they are still calling 10,20 etc. endpoints all doing work.
4. The increased number of API dependencies makes the page less reliable. How do you test what happens if that 1 specific API endpoint is down? I have seen apps appear to lock up completely because something didn't work. I have seen spurious and sometimes unreadable errors displayed.
5. What support can you get when the FE app doesn't work? Restart your PC? Try another device. Delete *all* of your cookies and try again? I have even seen people suggest reinstalling Windows as a solution. How is a BE app better? A single point of entry, error logging on the server and probably enough information to work out what happened.

Anyway, this isn't a post about how FE apps are bad, it is a case of how Devs have too much influence on businesses and therefore have lost their way.

## Developers have lost their way
In most cases, a back-end app is better but it isn't cool or sexy or whatever so Development Teams that lack experience leaders will go down that route instead regardless of the mess it casues. "But we don't do that" might be true where you are but there are seemingly very few apps that I use day-to-day that I find either enjoyable or at very least unobstructive.

When I have used the touch-screen at McDonalds and the screen registers the touch (the mouse pointer moves) but it doesn't do anything, that, to me, is a 100% critical bug with your touchscreen/software/OS whatever but yet, years after they have been introduced, these and other touchscreens do it consistently. Too hard to fix and not interesting enough?

What about UX? Selecting "cash no receipt" on an ATM to then get asked, "Do you want a receipt with it?" NO! Because I just selected "cash no receipt". Do the people who maintain this software not care? Are they not allowed to make changes? Is their build and deployment system so broken that it is not possible to do updates?

A particularly bad example the other day when buying a new mobile phone contract and a dropdown list on their app which loads all contract types in the list took, wait for it, 15 minutes to load the entries. This wasn't unusual according to the Manager. It then took 10 minutes to submit. What could possibly be happening for all that time?

## It's not just Developers though
Designers are similar. Design is fun, fresh, artistic so who cares about the fact that the dropdown list is in an awkward position, it looks good. Who cares that I have to dart all over the page to click different buttons, that's what Google do!

Product Managers can be your best friend or your worst enemy. They are never really incentivised to just improve the entire experience. Almost always, the product is broken down into small independent pieces that will lack consistency and will mean that your app always gets slowly worse instead of slowly better.

## Imagine if it was like this in Hospital
Could you imagine a surgeon who was terrible with patient interactions because they prefer actually doing surgery? Or a process to be admitted into hospital requiring you to walk to the 5 corners of the hospital site one after the other? Could you imagine if you were booking calls in and some of them simply got lost without anyone knowing or caring why?

There is a little of it in the UK because the NHS is such a large organisation but by-and-large, as Health is treated like a Professional industry, not like the original doctors who were random people who decided they wanted to be doctors, we have worked out how to optimise things quite well. You need training, you need to think about efficiency to get waiting lists down and all healthcare staff do training on "customer service".

## What should we do as an industry?
I think we should all be asking if we are proud of the industry we are in. Are we OK with people being paid all kinds of random amounts depending on who they know or where they work? Are we happy that most people's experience of software is negative? That touchscreens don't work properly? That support consists of asking people to try it again?

Are we happy that we are turning out brand new desktop apps that run on some kind of weird web browser thing that immediately starts breaking down with any kind of network disruption? Are we proud of the fact that we seem to release a new version of C# every 6 months even though we can't even get the basics right?

There isn't an International Order of Technology who can make these things happen so it is up to all of us to model it until it becomes normal. We need to start insisting on minimum standards of training; of correct application of project management; of what is acceptable in terms of security and then fight our hardest to be the best Developers we can be producing the best products we can so we can get the recognition that we currently expect but mostly don't receive.
