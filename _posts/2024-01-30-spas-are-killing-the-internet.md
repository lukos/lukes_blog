---
layout: post
title: SPAs are killing the internet
date: '2024-01-30T13:57:00.000+00:00'
author: Luke Briner
---

## What's wrong with the internet?
Well lots of things are wrong with the internet. Trust is hard to establish, crimes are committed, people are named/shamed/blackmailed online and other difficult problems but one problem is not difficult but has become that way and that is the reliability of online applications.

You know the sorts of things:
- You click a button and something doesn't quite look right, you don't know if the action was successful
- You do something and get some abstract error, perhaps in a toast message
- You are using an application that seems like it should be easy e.g. book a Doctor's appointment, but for some reason, it is really complicated, it's performance is terrible

In fact, I would go as far to say that today, if I use an application that just works as-expected, I am actually surprised because most apps are not like this.

## Why are apps so bad?
Of course there are various reasons but I am going to make a bold statement that most problems with most applications is the industry's obsession with Single Page Application frameworks or more specifically, an application that consists of front-end code in Javascript driving an API backend. Most applications do not need this complexity so why does everyone do this?

## The promise of front-end frameworks
- Much more UI-rich applications. This is a lie for the simple reason that all front-end frameworks actually do is html, css and Javascript so everything you can do, you can already do with any web framework
- More responsive. Another lie. The only time this is true is in the most simple case where you are doing some fairly lightweight transient changes like folding things up and down, hiding and showing etc. As soon as you add an API call, you either need to wait for it (if it is important) or otherwise continue whether it failed or not. Most responsiveness is easy enough to achieve with a backend framework + some basic JS/CSS using a lightweight framework.
- Your backend devs can also work on the front-end: everything is Javascript. Again, not true. Most of our Frontend devs know very little about hosting and databases so even if we did use Javascript on the backend, we would not benefit from this proposed advantage. We are not short of backend devs so we don't need people to work in both. In fact, some of our C# devs can do a reasonable amount of Vue js so it is possible to know more than one language.
- The richness of the library eco-system. Richness or the vast reams of mostly defunct or unmaintained packages? There are plenty of great libraries out there but how do you find them? Google? Stack Overflow? We have chosen certain libraries that have lasted about a year before the maintainer gives up for whatever reason. Why don't we just develop our own libraries? Ever heard of DRY? We don't do it because others have learned more than we will.
- Removal of workload from the backend. Again, this is only true in a VERY small niche of applications. I can accept that people like Facebook or Google with stateless servers can squeeze worthwhile gains out of this benefit but our system of 3 instances of backend applications behind a load balancer is not even struggling with 10K concurrent users. How many of us really need this? Also, it takes a large amount of discipline to do this since your APIs are having to do work, albeit potentially without processing state, but the backend will have to validate most of the same things that the front-end also has to so not much benefit here.
- Easy to knock up new pages based on the existing controls. Ha. Maybe in the most simple scenario like a listing control but in most cases, every page has its own challenges, its own edge cases that require new components or adding even more complexity to the existing components. Honestly, I could knock up most of this stuff with jQuery and a Razor page in a 10th of the time it takes in FE code. I also don't need to wait for someone to create me an API endpoint that might or might not return exactly what I want.

## So should we really just use a backend framework?
OK, so front-end frameworks don't deliver most of what they promise, but does it really matter? I think it really matters. Honestly, most people should stick to .Net/Java/Ruby/Python or whatever language you like since most of the best frameworks in these languages are already great.

- Try debugging a front-end app! I am porting an older app over to Dotnet Core right now and some things don't work. I can run up the app directly from Visual Studio, I can look at the rendered page and because of model binding in the request, I can see if it should work. If not, I can debug into the Action and see whether the parameters are populated. If not, I am likely to get some error or I can look at the raw request: "Oh yeah, unchecked checkboxes are not sent in a form". Do the same thing in the front-end? Good luck. The data probably hits at least 10 files in its journey, any of which might not work and you don't get great debugging in Javascript without peppering your code with debugger statements or console.logs.
- Is webpack a great thing? Webpack or whatever you use to bundle your front-end app is REALLY complicated. So complicated that I reckon most people probably copy and paste previous examples and tweak them to suit because, again, crazy numbers of packages and loads of options. It is clever, for sure, but it is not nice.
- What about yarn install and build? Our app only has about 3 JS heavy pages using Vue components but the build and install adds around 10 minutes to our CI build. The .Net app builds in about 30 seconds. Everything about these tools is horrible but no-one has been able to fix them, eventually someone else just say, "Yarn? Ha. We all use Vite now" as if they have fixed the underlying problem and haven't.
- I reckon 75% of our problems in production are related to the massive complexity of front-end code. You want some nice UI dependencies? Yeah easy. Oh wait, this breaks that page and the scrolling doesn't work properly any more etc. They also take much longer to debug and fix on average. Again, too many moving parts in front-end frameworks and most of these are just duplicating a lot of what you already have in the back-end anyway.
- You can actually get great response times in a back-end framework too. You can use Ajax to update a single element on a page instead of reloading everything. .Net had this in Web Forms 20 years ago and it worked great because the browser didn't have to repaint everything. We are seeing single digit ms database access times and page rendering in roughly 100ms. Is that really that bad compared to how snappy JS is supposed to be?

## What am I saying?
Simple. I have worked on roughly 10 major (public facing) applications over 20 years and none of them needed front-end code. Front-end code is ultimately just rendering HTML, CSS and Javascript so it cannot do anything magic it just pretends it can.

The joke is that people are already talking about server-side-rendering of front-end code because of these problems! You couldn't make this up.

Maybe in another 10 or 20 years, people will start to evaluate technology based on pragmatism more than idealism and start to see that in most cases, a decent backend framework is all you need!