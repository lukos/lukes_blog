---
layout: post
title: Perfection is a Software Engineer's worst enemy
date: '2024-06-22T16:02:00.000+01:00'
author: Luke Briner
---

## Perfection or Pragmatism
I have heard the quote that the correct people to employ are "smart people who get shit done" attributed to a number of people but I first heard it when reading about Stack Overflow (I think!) so I will attribute it to Jeff Atwood and judging by the search results in his blog, it's certainly a phrase he uses a lot.

Now I have worked with both extremes: People who are super practical but lack the intelligence to "work smart" and perhaps produce a long-winded or inefficient solution to an already solved problem. I have worked with less people that are of the other extreme: super clever but lacking the urgency to get stuff done and out the door but still people seem to fall into one camp or the other. 

Some people, however don't fall into either camp: They are neither particularly clever or super-pragmatic but I think these are most people that I work with and have worked with. They are kind of reliable, they work on lots of similar Line of Business Apps and can achieve most things to an acceptable but not amazing level. They get stuff out the door but perhaps not as efficiently as they could do and they create solutions that are just "OK".

However, these other people still suffer from what I think is a Software Engineer's worst enemy: Perfectionism. It afflicts many other Engineering disciplines and probably other industries but I wanted to keep my scope to an area I am more experienced in and not blanket the entire world in my belief.

## Code Detail or Scope Creep
I think this perfection comes in both deep and wide forms. The deep-form of perfectionism is someone spending far too long, for example, polishing a Regular Expression to get every ounce of performance out of it even though it is only used on the sign-up form. You know, someone wanting to save 2 ms on a form that is only posted once per customer and takes a second to actually execute. It's also like articles on the "right way" to write your CSS even though in my many years of software, I have yet to find someone who has failed because they used pixels instead of rems or ems or whatever else some purist believes in. Most of the time, UX sucks because it hasn't been thought about properly, not because the CSS didn't fit a paradigm.

The "wide" form of perfectionism is when something that should be simple and quick turns into a wider and wider stab at covering every possible combination of everything "just in case". You might have heard of the YAGNI principle i.e. You Aren't Going to Need It, but I also see lots of people who don't understand it. The counter to Yagni is "While you have the hood open..." but, again, we spend weeks or months building out things that haven't been tested with our customers yet. I see apps like Slack and Teams releasing new super-complicated features that I struggle to imagine are going to be useful to any more than a tiny percentage of users but have now broken all of our UIs because every change is a breaking change.

## Where to draw the line
Of course, if I say, "you're taking too long", "you're making this too complicated", your argument might be "I need to do it properly" so how do we articulate what is or isn't the right level of detail or abstraction or scope?

> The right amount of work is the minimum work I have to do to solve this problem in a way that performs appropriately well and is maintainable if we need to come back and make further changes

But the "OCD" kicks in easily: "You can't just use an arbitrary 'if' statement to check for a specific account number!". Why not? The code does what it says on the tin probably with a comment explaining why that account behaves differently. "What if another customer needs it?". If they do I'll either come back and refactor it or I will just add a second 'if' statement.

This is not saying that you don't care but writing something that has not been abstracted is not lazy or lacking due care "just because". If that is a quick and easy win for a customer to do something, deal with it!

"You can't just use a string comparison, it is much slower than a regular expression". If you can easily do the same thing with a regular expression, great, knock yourself out - being pragmatic doesn't mean you can't do something well by default but if that regular expression is going to take you 2 hours and the string compare 5 minutes, get over yourself *unless* this is code that is called 1000s of times a second and needs to be slick, then the appropriate level of performance is different.

## Business is business
The world moves very quickly. The software industry changes quickly. Most of your code won't live much longer than 10 years, if that, before someone decides on your behalf that Java is rubbish and Python is good; or server-side is 1990s and front-end code is the way ahead. Of course, all of these are nonsense because most frameworks can handle most scenarios without any particularly sticky problems so you are better off choosing what your staff are experts at and my own opinion that server-side apps are MUCH easier to reason, to debug and to maintain than any front-end frameworks.

Your business exists to pay bills and salaries and ideally, make enough money to make everyone's life a bit nicer. Your business succeeding might be a nice pay-off when the company is bought. It might be nice offices, nice free coffees and fruit or the latest hardware to work with. Every time you get stuck in some kind of "only 100% is good enough" trap, then you are eating away at that success and taking away from everyone in the business for a piece of code that no-one cares about.

So get over yourself, stop fretting, and just get stuff done!