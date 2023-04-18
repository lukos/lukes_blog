---
layout: post
title: The problem with customers
date: '2021-11-06T10:43:00.000+01:00'
author: Luke Briner
tags: 
  - Process
  - Requirements
---

I like pondering the bigger questions in software engineering, how the business works, which processes are effective and which aren't. I was reading the other
day about ["Stockholm parents built their own school app, then the city called the cops"](https://www.wired.co.uk/article/sweden-stockholm-school-app-open-source) 
and although the main point of the article was, why would the city not embrace the objectively better open-source app instead of taking legal action, I was more
interested in a frighteningly common problem, how can a customer with such a large budget produce something so terrible?

## Cheap means poor quality?
I think we have a built-in bias that unless you have enough money, you cannot produce something of high quality. Taken to an extreme, you could say if we don't
have enough money to pay a designer, our UI is likely to look very poor. If we cannot pay a team of testers, our product will behave poorly and perhaps be full
if mistakes.

The mindset is wrong. I think we should always talk about how much you can get for your money because quality should not depend on cost, quantity should.

> Quality should not depend on cost, quantity should

## Quantity instead of quality
What I am saying can be easily understood if I think about employing a decorator to decorate my house. Imagine I want gold leaf and plaster cornice and maybe
some nice paintings and heritage paints. She tells me it will cost £100K. Hmm, I say, that's a bit much, I only have £20K. What will she do? Well she might walk 
away but otherwise she will say that I can, instead, forget the golf leaf and plasterwork, have some basic paintings and normal paint and I can get that for £20K.
What she doesn't do is water down the expensive paint, apply gold leaf in a really patchy way to save gold and use some out-of-date heritage paints. What I get is
still high quality but I get *less* for my money.

## That wasn't the problem
Ignorning that metaphor, though, this clearly was not a problem for the City of Stockholm, spending an eye-watering $117M on what is basically a database with mobile
app access. So in this case, even if you do want world+dog in your system, surely 117M is enough money for that? Let's say that half of that money was for hardware, that would
still leave you with 58.5M, with which you could easily pay 20 of the world's best developers $1M each to spend up to about a year producing this.

Somehow that didn't happen. They spent more than double what the Royal Society for the Prevention of Cruelty to Animals charity received for the year 2020. A UK organisation
that employs 1420 employees and dealt with 140K welfare incidents in 2020. Twice as much for a school app that is, at best, a convenience rather than a step improvement.
It makes me shudder thinking about it.

So how can a customer like this spend so much and get something so bad?

It's not as weird as it sounds.

## The birthing of an idea
You can imagine sitting in a meeting at the department for education and talking about various challenges. "I know, we should create a system to manage all of these problems
and provide a *unified* system that will obviously solve all of our problems". You can imagine everyone nodding in agreement. Of course, this will solve all of our problems.
What better way to solve our disparate and aging systems by creating "one system to rule them all".

At this stage, the idea has value in itself regardless of the fact that no-one has any idea how much it would cost, how long it would take and whether what comes out the other
side will be much better than what you already have.

Someone will ask though, "how much will this cost?", and because there are no official requirements, someone probably rings up a friend who writes software and asks them very
"finger in the air", what are we talking here? 50K? 100K? 1M? More? If you have ever asked an engineer how much for App + API, they will probably think, "that sounds like about
a years worth of work with a team of perhaps 10 people so maybe 1M?".

1M? Great, much less than we thought (not the price has now moved from ballpark based on no details to budget price). We can get authority to spend 1M. To be honest, if it was less,
it wouldn't seem like the right kind of system, we need to spend our large budget right?

## Ideas are free - implementation is everything
I think Paul Graham coined this phrase but it is so true and so under-appreciated. It is easy to suggest a "system" to unify other systems, to create an "app" that parents can use
that can solve a handful of current pain-points. It is only, however, when you start getting into the details you can tell whether your staff are actually intelligent and are able to
handle what engineers have to constantly handle, compromise. "Yes Mrs Customer, what you are asking for is fair in isolation but to add this to the existing system makes it very messy
and will reduce the usability".

If you are lucky enough to have a very intelligent, empowered Project Manager on the job, it might not be so bad. The Supplier can work with the Project Manager and talk about how to
get to MVP as soon as possible. To do the design work up-front and not wait until something is produce before changing it or adding something else to it. If you need something like
an example data dump from a database to build the "import function" then you want this early on. What you don't want is to find out that it changed 6 months later and you now have to
rework (not rewrite!) all of that code you already created.

In most cases though, not only do you not have an intelligent or empowered Project Manager (they might be one and not the other) but you often don't even have a single-point of contact.
You have meetings with 10 different people who all have their own opinions, their own priorities, and no-one drawing them all together. No-one who can say, "if we are going to add that
we need to redesign the main menu so no!". You are likely to get staff turnover, especially in this case where the project was started in 2013 and took 5 years to complete.

When one of those people has sent you broken data *again* but there is no-one to beat them into shape, what can you do as a supplier? You can push back another 2 months and you would expect 
someone to get angry at the extension but most of the stakeholders are not paying from their budget, they just want their objectives to be met.

## The devil is in the timing
Just to add to the rub, testing, something that should be a non-reducable element is often shortened because everything else has taken so long and it is starting to get embarassing. The
testing that might find those really icky parts of the system are not done. Even worse, high-level testing that proves that the system is consistent and usable is not done until the end,
then what happens? More rework?

The worst of all is the idea of the final "sign off" from a senior manager or the sponsor. This is usually done at the end once you are already late and have implemented everything. Yes,
we have had a senior manager before complain that something in the system isn't right the day before launch. Change the wording, move that around. Wait a sec buddy! If you want input,
you should have had sign off of the text earlier in the process.

## Just say no
There is a weird relationship between many suppliers and customers. Imagine if I employed a plumber and he wanted to run the pipes across the ceiling, you would say no, even if you have to
pay a bit more to run them on a longer but neater route. But imagine if the plumber said that he couldn't run pipes though a concrete floor and you should run them outside? You might not like it
but you can understand that to run them through the concrete might cost thousands and take a long time so you compromise.

What happens in software? You have people are not qualified, not experienced and sometimes not even intelligent in charge of decisions for a multi-million pound system. You are a professional
and you object to a decision that make the system much slower/less usable whatever. Most of the time, the supplier loses, the software suffers, the customer gets away with it and someone
has a very large bill. Of course, the bugs that get introduced by such terrible changes are always the fault of the supplier even though you warned them.

## What do we do?
What is surprising is the level to which these bureaucratic messes overspend on projects. What would you do instead? They weren't to know that the Swedish parents who created an alternative app
would have done better and ironically, if they had been employed instead, they would likely have produced the same mess or walked off the project because the supplier was not likely to be the
problem.

The reason it worked was simply that someone produced what was *needed* and not what was conceived. We all do it when we create software. We cannot wait until people are asking for something,
we pre-judge "of course they will need X and Y" even if they don't. If the customer does need X and Y, they are more likely to prefer just doing ABC sooner and waiting a little while for X and Y.
We need to realise that most people use a tiny fraction of software functionality so getting more comfortable with agile delivery and the fact it is OK to deliver a very basic function now
and add better stuff later. "Parents, you can currently see if your child has Phys Ed today but next month, we are planning to release a whole calendar feature". Release, measure, improve and
keep your product simple.

Lastly, we need to be able to say no to customers. "What would be good (i.e. for me) would be a feature that does blah", "Sorry Mr Smith but we don't see a lot of value in this feature for most
parents, perhaps you could use this feature instead".

Maybe someday someone will write a good book like "Don't buy software until you have read this". Maybe I'll write it. Probably I won't.