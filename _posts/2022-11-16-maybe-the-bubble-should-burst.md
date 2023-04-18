---
layout: post
title: Maybe the bubble should burst
date: '2022-11-16T12:24:00.000+00:00'
author: Luke Briner
tags: 
  - Business
---

## Twitter is firing loads of people
This weeks technical headlines are mostly this. Also Meta, Amazon, etc... all firing lots of people. This shouldn't be a surprise by any stretch but people's tweets, comments and news articles just remind me that for so many of these people, their bubble is so thick that they are genuinely suprised/angry/upset.

We can look at Elon Musk and say many insulting things but what is happening at Twitter is not really that different from what is happening at many other tech companies. Investment is drying up and companies have to make money. The fact that these companies have employed 1000s for years and years on wages that are somewhere between very generous and obscene, is itself obscene.

## Outside the bubble
I work in Tech. Senior Management, decent salary by UK standards but less than £100K. I work at a small company with a tech team of around 12 and alongside a Dev Manager am in charge of the tech for our company, so I'm kind of important. For me, I wouldn't dream of hiring someone that I could give a full-time set of work to just because the company has cash in the bank. I am not employed to build my kingdom but to support the business. If the business wanted to introduce some new product which meant more servers; or if they wanted to increase formal support to 24/7/365 which meant on-call rotas, then part of that cost would be extra people. Currently I have one SRE who helps with monitoring and logging and an apprentice.

This is normal life. I like my bosses. They are not perfect and neither am I. They sometimes make decisions I don't like/100% agree with but I support them because I believe in the company and their vision. Because I support them, they trust me to implement their product decisions in a way that doesn't break what we're doing. I don't adds tonnes of metrics to the product if that affects performance. If there is no way to do it without performance suffering, it doesn't get done.

This is how most people live, although 99% of the world's population get paid significantly less than I do but I get paid significantly less than people in the Unicorn companies.

## Inside the bubble
People who get a job at a big unicorn usually think they have arrived right? What could be better than working for some of the most famous names in the tech world. You have offices that are so cool, all your food is free. Some even have laundry and gym services. Is this because they can afford it and you want to take the job there? No, of course not, you would take the job anyway on the wages they pay you and the brand name. They do it to make you stay in the bubble as much as possible. To drink in the hype and branding so much that you forget what the outside looks like.

Watching Apple release conferences makes me feel nauceous. Look at this new camera: everyone claps/cheers/faints/cries etc. and if they don't, they get told to from the stage! I'm thinking, well-done guys, your team of 400 engineers managed to build a camera, which is what you are trained to do and paid for.

## The tech though!
Now, culture and cultishness aside, I guess if what they produce is so good, then maybe it's OK right? But is it? Is Twitter really that different as a user, than it was 5 years ago? Maybe for the professional users it is but honestly, I find most apps including Twitter, Facebook and occasionally GMail only get worse over time. Microsoft products? Same. Click a button, web request takes place, network glitch, etc. nothing. Is that what all these so-called "x10 Developers" produce?

Elon Calls out the "death by microservices" mess at Twitter and actually, he's right. I said it. We can all excuse, "but we were told we had to etc..." but if the so called "best engineers I've ever worked with" couldn't produce the functionality that was needed with just basic good-engineering. Fearless refactoring. They certainly had enough fingers on keys, they could have ported it between a number of languages in not a very long time. Nope. We instead have basic operations that spawn a gazillion asynchronous requests that by definition will never result in a good experience for a user on a slow network. Jumping pages. Infinite lists that have no more entries. Find results that are all over the place.

My opinion, which I expect is shared by many others, is that most software I use today is no better and sometimes much worse than it was 10 or even 20 years ago. It sometimes has some better features but at a significant cost, almost always related to the addiction to everything being API-driven and online-first.

Visual Studio 6. My first IDE at work. I never remember any slowness with the UI, startup etc. It never locked up on me. The only slowness was build times of large solutions because of slow PCs. Visual Studio 2022 like its more recent predecessors is slow to start, slow to operate and can take a minute just to parse a few hundred files to build an intellisense database. It does not seem to build noticeably quicker than VS6, the UI is clunky, even the search in the MRU list can take 10 seconds! When you look at it, it runs like 10 separate processes including frightening things like node js servers just to run what is basically the same app.

So no, the tech is not good. It is the same old crap we all have to maintain at our companies. The only thing is, the swagger that some of these guys at big name companies have makes me think they should be much better than me. All these, "we invented gRPC", "improved .net performance by X", "Senior engineers on $400K" etc. and the result is embarrassing.

## How is it?
How do these companies manage it? Well we probably all know that money talks right? You can be the crappest company in the world but if you make money, investors give you money to make them even more. Many won't care how efficient you are, whether you are over-paying people etc. The assumption is to splash the cash, get the "best" people and you will create stuff that is so good it will sell.

So you get this weird incubator environments while the rest of the world is still trying to maintain their ASP Web Forms projects, you are already inventing .Net 7, patting each other on the back, telling the world how clever you are that you added some new language feature and made it a bit faster and laughing from your ivory towers at us peasants who are cursed with Web Forms. Bugs from years ago never fixed and never will be. To you, everything is greenfield, most theoretical and always ready for the latest and greatest.

## schadenfreude
I will apologise because as I read people writing things like "After 10 years at Twitter as SRE, I've been sacked", I don't feel sorry. No doubt you got paid a packet on top of great benefits and shares in the business. You were in a team of how many hundreds and between all of you, most didn't really add any value to the business other than what most people have to do on a quarter of your salary at other businesses.

I don't care that you think you are brave calling out Elon or Zuckerface or Bozos as heartless or incompetent because the whole hermetically-sealed self-congratulating environments you work in are already farcical. They were farcical before you arrived, while you were there and will be after you leave because that is how they operate. In fact, if they were less farcical then maybe they would have already failed because people would realise they are built on smoke and mirrors and will crumble down at the smallest problem.