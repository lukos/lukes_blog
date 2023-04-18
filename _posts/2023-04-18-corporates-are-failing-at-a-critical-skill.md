---
layout: post
title: Corporates are failing at a critical skill
date: '2023-03-18T19:37:00.000+01:00'
author: Luke Briner
tags: 
  - Applications
  - Mergers
  - Migration
---

## How bad can an application be?
We have probably all worked on applications where something takes a few seconds when it feels like it should be pretty much instant, where we have struggled to eek a few more processor cycles of efficiency into some code.

Imagine if you had an app that needed to populate a dropdown list with all of several hundred product codes, how *long* could you make that take without adding artifical delays? Evena massive table scan in a database could probably complete in less than a minute right?

What about 15 minutes?

Yes. I was visiting an EE mobile phone shop and their system couldn't lookup the product code I was sold so they had to use the "manual" way of getting the list of everything and picking the correct one from the list. It took 15 minutes to populate the list and another 10 minutes to submit the selection. And the list wasn't in alphabetical order so the poor sales girl had to look carefully to match the alphanumeric code.

## What the Manager told me
The Manager was helping the sales girl and was obviously quite IT savvy. He had previously worked for O2 who had "great systems" but EE, like many businesses, decided that tablets are way cooler than PCs because they are portable and stuff, which means, of course, they are on a VPN running on a shared wifi network and tablets have NEVER been good at high throughput usage. Have you ever tried typing quickly on a tablet? Even on a bluetooth keyboard it doesn't work.

But it gets worse. EE, like many large businesses, has bought various companies, merged with others etc. and the unwelcome outcome of this is inheriting completely different administration systems. For reasons that I cannot comprehend, they are never seemingly interesting in porting any information over to their new system so they kind of glue them together with horrible applications that call into other apps in other locations and probably glue the results together in XML or something like that.

So what were they doing about it? I don't know what is worse, making a customer wait 30 minutes to choose an item from a list or paying 1000s of employees to sit around waiting for crappy slow systems to do their job? They even had this identification system (presumably not hosted by them) where you take a selfie and upload it with a picture of your driving licence. Same thing, took about 2 minutes for each step of the process - I guess it could have been terrible WiFi in the store, who knows.

## The big rewrite
Well, then I heard the line that made me die a bit inside, "They are planning to replace everything with a new system then it will all be OK". You might have heard of "The Myth of the Software Rewrite". Many articles have been written about it but there is a fundamental disconnect between what *seems* like being possible and what *is* actually possible. 

For a start, even a single page/action/component in a single application has 5+ years of knowledge in it, not just code but knowledge. Seemingly unimportant details like why this part is called before this part. Why something weird was done with the authentication (intentional or a bug?), is there code there that is wrong but it is wrong because it works? Is this logical clause still relevant or just messy code? The fact is even a single entity would take a day to understand and that's assuming you have the original designers/Developers/Managers available to answer the questions.

Secondly, there is always the temptation to "just improve this" while migrating. Seems reasonable but that "slight improvement" is hours of arguments, discussions, designs, problems with new library/framework X, build servers, deployment issues etc. Something which could be migrated as-is in, say, 1 month, takes 6 months.

Thirdly, who is going to do this work? The Development Team are the only ones who are possibly suitable for the task so who is going to do business-as-usual on the existing systems for the, perhaps, 2 years+ it would take you to rewrite if you were working flat-out?

Fourthly, when are you ready to deploy the rewrite? What if you spot a major bug after releasing? Do you immediately revert to the old horrible app? How much testing and testing time will satisfy you that the new system is now ready to go and will be fixed forward? Some organisations test each release for a month, how long would it take to test every part of the system? 

And what happens when the needs of the organisation change midway through the rewrite? What happens as people leave and others start? What happens when the Manager whose stupid idea it was in the first place decides to pretend to move on to "new challenges" and leaves everyone else to carry the mess they didn't want until the replacement Manager decides how rubbish the idea was and that they will rewrite in this even *better* way.

The rewrite in most cases is complete nonsense suggested by people who don't understand. If you are recruiting a CTO, ask them how they would solve this scenario. If they say rewrite, do NOT employ them.

## So what?
So what is the critical skill that corporations lack? It is the skill of migration. Not rewriting an entire app but moving pieces of it at a time into a new format/language/framework/whatever with a proxy or an application acting as a proxy sitting in front of it.

I first appreciated this when I read about Microsoft suggesting the ability to migrate e.g. a .Net framework app to Docker without rewriting it. How? By taking parts of it and directing those paths to the new app and slowly but surely moving everything across. Much smaller pieces of work, much less risk, much easier to test and much easier to release and fix if there are issues.

We mustn't feel the need the get to the finish since Industry changes more quickly than our migrations take but we should be in a place where the stuff we need to change/improve is in a format we can work with and we should endeavour to keep moving but at the point we then inherit a new system, we then need to be able to pull that new system behind the proxy and allow its features to also be moved invisibly to the user.

And we should definitely use cache for dropdown lists that take 15 minutes to populate. EE!