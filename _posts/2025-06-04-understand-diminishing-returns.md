---
layout: post
title: You need to understand diminishing returns
date: '2025-06-04T13:39:00.000+01:00'
author: Luke Briner
---

## Rules for Developers
I don't really like those articles about "the top 10 things you need to know as a Developer" because like a lot of things in life, it sounds sensible but really it should be "the top 10 things I needed to know with my background, training and experience that are probably very specific to me".

Things change and lessons that were once hard-learned before are not really a thing any longer. Back in the day, people would lose their voices telling you not to write your own session or auth system for a framework. Why? Because it was hard to get right and...a lot of early frameworks didn't have them built-in. Guess what? They do now so very few people need to learn the lesson about using the build-in features of a framework.

Anyway, one thing I have learned that I think a lot of Engineers are prone to get wrong is understanding about "diminishing returns". Hopefully you understand the phrase but imagine you buy a cheap second-hand car and you want to make it run better. You would start with the cheapest easiest things right? Maybe clean the fuel system out, replace the filters and spark plugs etc. If you want even more performance, you would move on to slower more expensive things like rebuilding the brake calipers to make sure they are not binding. Replacing wheel bearings and piston rings maybe. If you were still not satisfied, you start getting to the very expensive options like replacing the gearbox, replacing the engine, re-tuning it, replacing the exhaust, the inlet manifold etc. These things don't necessarily make your cheap car a lot better and start to cost so much money, you would be better off buying a faster car and not spending any money on that.

## Diminishing returns in caching
When Developers learn about caching in web apps, they usually get excited because that database query that used to take 100ms now comes from cache and only takes 1ms - at least it should. However, you realise that you need to decide how long to cache something for. Cache it for too short a time and you risk excessive database calls to refresh it, leave it in there too long and it gets stale and might cause your customers some confusion - how should you balance it?

If you already understand that you need to balance it, you probably don't need to continue reading but there are lots of Developers who are on holy missions of perfection. Why cache it for 5 minutes when I could cache it for *24 hours!* Just imagine all the database calls I would save.

These people are a problem to have in your team and will cause far more issues than they solve. Why? As we already said, there are diminishing returns and sometimes the savings are not even there in the first place!

It is critical that Developers measure things and we often don't. Cache = better than database calls right? Well, not necessarily. It depends how you deserialize the cache. Json? I thought so! Have you actually timed that? You might actually find that a database call materialised into an object is actually faster than using cache but that is not the main point so I will leave you to ponder that. Most databases now run on solid-state disks and are exceptionally fast. Our main application at SmartSurvey has typical database times of 1ms and that is with a synchronous replica! Beat that Json deserialisation!

Let's imagine that every call to your web application currently calls the database to load the features that are available on the user's plan. On a Basic account you can't do most things but on Enterprise you can. Great, we load a map of plans to features and when the user tries to access something, we do a check that the feature in-question is on the plan that the user has. Let's imagine that we have something like 10,000 calls per day. 10,000 lots of 1ms database calls is only 10 seconds of time in the entire day! Don't cache until you know there is a problem. But OK, imagine that we have 100,000,000 calls per day, now we have 27 hours of database calls per day!? In other words, we would have to be running multiple web servers but that is significant and would definitely benefit from caching.

## How long to cache for?
The perfectionist Developer thinks, well, we hardly change the features on the plans very often, so why not go nuts and cache them for 8 hours. If we do update the features, we can either wait 8 hours for the cache to refresh or we could simply re-deploy the application or restart the web servers and it will reload cache. The perfectionist is happy because we have now gone from 100M calls per day to 3 per day per web server. Wow, feels amazing. But we have created a significant burden for people planning to change features. They either have to change them 8 hours in advance (in which case they might appear too early) or do something else to reset the cache but...3 per day right!

Here's the thing. What if we only cached them for 5 minutes instead of 8 hours? That is only 288 calls per day. Maybe a couple of orders of magnitude higher than 3 but compared to 100M, still an enormous saving and remember, at 1ms per call average, only 288 milliseconds per day per web server and without the 8 hour lag between updating features and seeing them appear in the application.

## Understand your Developers
You need to identify this as early as possible. The perfectionist Developer invariably takes too long to develop their software, is over protective of it because of the time they have invested in it and doesn't usually have the massive beneficial effect that they think they do by spending all of that time.

The way to manage it is to ask for data. If someone wants to add caching (in this example), ask them to measure the problem - if they can't do that in 5 minutes then don't let them do it. Ideally they should already have the data before even mentioning it, otherwise how do they know it's a problem? You can then discuss if there is a problem and how far to go to make an improvement before they start work.

Having goals/OKRs/targets can also be really important to ask that person how the change will move any of those targets. If a page takes 1 second to load and the database isn't struggling to keep up, will the user even notice the 20 or 30 milliseconds you improved? No, they will not!