---
layout: post
title: We have forgotten that software is tooling
date: '2024-12-17T14:22:00.000+00:00'
author: Luke Briner
---

## What was the last great software you used?
You know that feeling when using software that "just works". There are a few things I like: SQL server is hard to setup, especially with a failover cluster but once installed, I have very few problems. Same with postgresql, yes
it can be complicated to configure but it still works and I guess as a database engine, it does have to be reliable.

The only other things I generally like are utilities like Beyond Compare, Notepad++ and Nimbletext. I guess it is easier to make something relatively simple work well.

What about the larger apps? Office, Visual Studio, MS Teams or those touch screens you get in fast-food outlets and supermarket "self checkouts"? I find most of them frustratingly bad.

Consider MS Teams. At its heart is an instant messager but, of course, companies can't leave things doing one thing well so now it does lots of things, some of them badly. Even setting up a chat is more tricky than necessary. Setting your status is fiddly: editing your status requires a button press instead of just clicking into the text box. Getting at downloaded documents is weird and usually takes me two attempts because I don't want a preview. I often download things multiple times by accident. Task management is icky, updates are weird, code editing is tricky too. I even have something that happens occasionally when I paste some text into the box and it immediately gets deleted and I have to do it again.

Now let us think about that for a second. Your app, which is designed to send mostly text to other people has some kind of functionality that (presumably incorrectly) is deleting something in the very box it is supposed to be sending. Sometimes, I even have these undocked windows appearing, which I have no idea why or how. Perhaps the keyboard management is so slow that it detects incorrect key combinations.

I pick on MS Teams but loads of apps have similar rubbish behaviour. We have web apps that we originally accepted were slow and unresponsive because of 128K dial-up internet access but that is such an old problem that we should be seeing super slick apps but we are not. A big culprit is the addiction to using front-end frameworks for functionality that would be much better served as a normal server-side application. A single page request is much faster than 10 API calls but we insist on using them anyway.

## Software as a tool
Have you ever seen the UI for Salesforce? Incredibly bland and slightly inconsistent but guess what? They have over 20% of the world CRM market. How comes? Because their software is built as a tool, not as a plaything for Designers, Product Owners and Front-end Developers. I don't have much experience with it but when I've seen others using it, each page loads quickly, it doesn't try and create abstract UX, it is mostly form-based because that's what you are doing.

Software is supposed to be a tool, not a vanity project.

I saw this old documentary from the 1970s about electronics and they were talking about how electronics had enabled computer systems to be available to many businesses. One New York bank bought a system for $7M and you know what it did? It had a single screen showing how much foreign currency was held by the bank so that the traders wouldn't over or under buy it. That was it. Why was it $7M? Because previously, when someone bought or sold currency, they wrote it on a piece of paper and every few minutes, someone would collect these pieces of paper and take them to a central office where they were all added up, the balances updated, printed out and taken back out to everyone. Needless to say the latency could cause the bank to buy or sell far more currency than they wanted and the $7M was deemed very reasonable as a simple business decision.

What would happen if that same system was built now? Probably a web app (fair enough, it's probably the easiest way to distribute such a thing). The architect wouldn't choose postgresql because "vacuum" and would instead wire up their own hybrid of something. They would obviously use a front-end framework like Vue or react because only stupid old people write server-side applications and "think of all the reuse and flexibility you get". I've written before and won't revisit how many reasons for using front-end frameworks are myths. We would probably add a load of unnecessary management tooling, and if you work in the Visual Studio Team, will probably spin off random node js and msbuild processes for some reason.

## How do we get back there?
It is hard, I think, because unless the Developers are highly respected in the organisation, you will always have some Business person seeing some "AI" or whatever the fad is and wanting that in the app, even if it won't fit easily in. The pressure leads to design fragmentation, to technical debt that will never be paid back, to front-end frameworks with all of the complexity they bring and maybe even splitting into 100 microservices, all with their own foibles.

Each of those decisions is more touchpoints, more risk, more opportunities for bugs.

On the other hand, Developers can be their own worst enemy. I have worked with people who can implement something complex in a relatively simple and easy-to-maintain way and others who quite simply would happily bolt something on.

We are completely spoiled with tooling and knowledge so we should be able to do this. Visual Studio should be able to add new features without spawning node processes and random msbuilds running in separate processes even when you are not compiling anything. Teams should be 100 times more snappy than those Windows apps we had in the 1990s and people have to stop that fascination with everything being online and calling home. No! Your APIs are NOT fast they take noticeable time and cause frequent lock ups. No! I do not want or need updates every 2 days. If you are doing that, you are either adding too many features or your code is too crap that it needs fixing all the time.