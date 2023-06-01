---
layout: post
title: UX is still done badly
date: '2023-05-17T09:48:00.000+01:00'
author: Luke Briner
tags: 
  - UX
  - Performance
---

## I have a gripe with most UX processes
Any of us who work in SaaS know how much we talk about User Experience (UX). It encapsulates User Interface (UI) but also includes everything else that affects how easily someone can accomplish their task on your software. If any of you work in this environment, you probably know where I am going!

In most cases, I would argue that most applications that I use will gain no value out of the constant fiddling and messing around with the layout of pages, the constant iterations that are supposed to lead to some Holy Grail of usability but which end up doing the opposite. The truth is that many UX experts do not understand the basics of what they are actually trying to do and are instead justifying an existence that is really not needed.

Let us look at some examples.

## Azure Devops
We use this for Task Boards and for Git repos. As an application, in general, it is fine although it suffers lots of the issues that web applications suffer. Let us consider some of the problems with it:

* Large mouse tracking movements to do basic operations. Go to a board and add an item and your mouse darts one way or another. Go to repos and create a branch, your mouse goes all over again. Google are partly to blame with their "material UI", which was imho overhyped garbage
* Frequent performance problems due to an insanely large number of ajax requests made on all pages. Some of these are ironically likely to be UX and tracking frameworks but honestly the biggest issue here is that the complexity creates all kinds of subtle edge-case bugs and a tonne of log noise

I think these are my main two complaints but note that neither of these requires a UX expert. Anybody maintaining the system *should* notice that the mouse moving all over is really annoying and makes it slow to do the things you do all the time (keyboard shortcuts on desktop apps get around this). I can't believe that nobody at Microsoft has ever fed this back but I suspect they would be told, "we can't take your word for it, we need customer feedback". Bullshit. You don't need customer feedback to know it is objectively wrong to make someone need to move their mouse all over the screen instead of concentrating in one area and adding keyboard shortcuts for basic operations.

Ah, but is this the most important thing to work on? It doesn't matter because it is bad UX period. If it isn't the most important thing then you are deciding that UX is not important since something that is clearly wrong (and shouldn't have been designed that way in the first place) now has to be endured by your customers. UX is the shop window to your application so if it is bad, your whole application looks bad.

The complexity of the page is another area where very poor architecture decisions cause UX problems that don't need expertise to decipher. The knock-on effects of this are not just performance but almost completely non-debuggable errors. If I see a console error, I am told to ignore it - I should never be told this! If I have a problem like I did where it kept referencing an old email address - I was told to delete my user and add him back in, regardless of the potential for massive disruption, broken history, all kinds of other unrecoverable problems. Why? Because someone didn't do their job properly, because they overcomplicated something which for the most part is a ticket management system.

## Windows
Any of you that use Windows regularly will have noticed that every single major release moves stuff in the control panel. We used to have Add/Remove Programs, then they moved to settings and I think we have had an "App" panel, then something else on Windows Server and then its moved again on Windows 11. None of these have ever had any utility because they all open the same thing, it's just each time, Microsoft have presumably worked really hard to make their new OS look different and each time, all they do is make you relearn stuff and cause every existing blog post to immediately become out of date (and there are loaaddds of blog posts about doing things in windows).

Ultimately, Windows has been a start button, apps and settings for years and years so why can't they leave it alone? Some UX person told them it would improve something by x percent so why not? We A/B tested it (pukes a bit) so we can piss off all existing users just because people slightly preferred something over something else that wasn't even broken. Do you know where you can use A/B testing? In only one place: Where you can't decide which *new* thing to go with, most of the time you should know because if something is simple and obvious, you don't need to ask your customers (or stakeholders).

## I mostly hate front-end frameworks
A lot of this came out of the fad for front-end frameworks. These fixed a problem that 99% of people never had but everyone jumped on board anyway. Why? Because we like fads. Software Developers mostly don't understand that the problems you have are not as bad as you think and the solution doesn't come at zero cost.

Who does need these? They are really useful for people doing *really* complex UI stuff. Azure Devops is a little bit complex but could easily have been a server-side app and would have been much easier to develop, debug and maintain. Facebook might enjoy being able to have components that update independently of each other but even that could have been rendered server side with some ajax code embedded in the front-end, although at least for Facebook it was good for a number of reasons: Any processing they can move to the browser makes a big (multi-million dollar) difference in their hosting costs; it makes the use of massive scaled microservices slightly easier but mainly because of the large numbers of people on lots of teams who can play separately without impacting everyone else too much.

Front-end frameworks are always more complicated and costly to develop and maintain but if you are Google or Facebook, you can both afford it and it does buy you *some* advantages.

Of course, for most of us, we don't need it, we can easily do the same thing in Java, Ruby, Python, .Net or PHP with a very clear reasonable structure (MVC or MVVC). The complexity of the FE framework then causes an unusual side-effect. We feel the risk, the slow pace of development, of the fragility of the the code with its uncontrolled interactions between 100s of components so we implement a much slower and stricter design process because we sort of need it "as right" as possible from the first deployment. God help us if we mess it up and have to refactor.

So ironically, the development of FE code is not just slow in itself but it causes slowness at other levels. When I develop something in dotnet core Razor, I feel none of the anxiety because a change literally takes a few seconds to implement and test and if it is a little wrong, it is easy to refactor or change it to make it work correctly. Do something in a FE framework, you might need to change a number of files, rebuild it (the FE build takes 5 minutes on our app) and hope you haven't missed a side-effect that will occur in production.

## The correct way to do UX
This might seem like something deliberately designed to poke the UX experts but honestly, if you create a product, you should pretty much know how to make the UX nice from day 1. You can decide to do something quite standard or invent your own UI but once you have established that, you don't need to A/B test everything, you don't need tonnes of customer feedback, you need to use the product yourselves so that you learn what does and doesn't work properly.

If a customer tells you that something really doesn't work well, you need to be confident enough to ask yourselves whether you simply messed up - no worries - or whether a particular customer has a different use-case and whether you care about it. A large failure of many businesses is trying to please everyone and losing focus as a result.

Classifying things can also help you to keep your decisions clean. Many suppliers are too quick to make an isolated change to something because they can and because they found that 5% more people prefer something else without actually considering what the product i sbeing used for. For example, the way Twitter is used by an individual vs a business is very different. If twitter tried to make the same interface work for both, it would become a mess. (As it happens, Twitter didn't really seem to do much for Businesses leaving it to people like Tweetdeck to pick up the space) but if someone with a good head identified that a Business is not saying that the normal interface is bad and needs changing, they are saying that it isn't designed for their use-case, you can then decide to a) Create a business interface or b) politely decline to make changes which would make the existing experience worse.

Know your product, keep things simple, and believe in yourselves. That is how you do good UX.
