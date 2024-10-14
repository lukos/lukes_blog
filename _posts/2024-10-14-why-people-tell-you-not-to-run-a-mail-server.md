---
layout: post
title: Why people tell you not to run a mail server
date: '2024-10-14T20:26:00.000+01:00'
author: Luke Briner
---

## Ask HN: How can I run my own mail server?
Ask this question and like most Hacker News questions, you will get a range of answers from "It works fine for me, just use X and Y" through to "Don't even bother, just use Amazon/MailJet/etc.". I don't like pithy answers or anecdata to justify a position. It works for you does not mean it will work for me and vice-versa.

## What is a mail server?
In most cases, we are talking about a Mail Transfer Agent (MTA), which is mainly involved in sending mail. You don't really get problems with incoming mail because providers don't mind sending you stuff, they just don't necessarily trust you to send mail to their users.

A Mail Transfer Agent is largely a sending buffer that can be configured to hold messages reliably across power-cuts etc and attempt to send them a number of times over a number of hours before failing. Why might email fail? Mainly due to the transient nature of the internet, flakiness of DNS and very often because people haven't setup their stuff properly! Will a retry work? Maybe, maybe not but I don't want my desktop PC to stay switched on to retry mail so I am happy to hand it off to my MTA and get it worry about things in the background.

## So what is the problem with running your own?
There aren't many high quality mail servers that you can just install and use. `hMailServer` is OK but Windows-only. I personally use Postfix at work and although it works great, it is not designed to be user-friendly and it has taken me a number of years to tweak the parameters to work well. Just the other day I realised that although I had the retry setting to start at 1 minute, the deferred queue check was only 4 minutes, which was causing a lot of backing up of queues when sending large volumes. That said though, I ended up with an Ansible script to capture this knowledge and wisdom and allow me to spin up new servers pretty quickly (less than 5 minutes from provisioning).

You need to ensure SPF, DKIM and nowadays DMARC are setup correctly on your domains although again, it doesn't take long to learn it and set it up.

The much bigger problem with running you own is that you are likely to get blocked at some point by one or more email providers and you get varying levels of support to resolve your problems!

## Why are they blocking me?
The ultimate answer is that they don't trust you but worse than that, you might have done aboslutely nothing to earn that lack of trust other than some "heuristic" that you look dodgy. Here are some reasons that I have been given about why my servers have been blocked that are not actually proof of anything:

* The IP address is too new. So what? Spammers spin up new machines! Yeah, well so do legitimate businesses!
* Your IP address is on a dial-up list! Is that even a thing in 2024? Yes, these are dynamic IPs that ISPs give out to customers so it looks really dodgy. Although, of course, they are also "elastic ips" that cloud providers give out as the closest to static IPs you will ever get. The other problem is that no-one knows the real use of IPs, they are bought/sold and transferred all the time
* You are sending too much mail. Well we send up to perhaps 500K emails per month on behalf of our customers so of course we will send a lot of email. That means exactly nothing.
* You are not sending enough email. (You can't make this up can you)
* Your sending pattern is erratic. Well it would be because our customers send emails when they want to send them, not when its convenient for Microsoft.
* You are spoofing the "from" address. Wrong again. Like many providers, if a customer doesn't verify their domain, they can only send from us but we will put their email address as the "name" of the email (like the "Luke Briner" <lukebriner@example.com>) so wrong again.
* You are rate-limited due to IP reputation. Wrong on both counts. Your own monitoring tools shows that we achieve very high quality sends with minimal abuse (spam) reports and you are not rate-limiting it otherwise our emails would still be going through, just more slowly

## What can you do about this?
Mostly nothing! Some error messages in log files give you an email address (Deutsche Telekom who are usually very good with unblocking requests), others point you to a page where you can request a delist (Proofpoint do this but you apply to get delisted and they NEVER reply to you despite claiming that they *really* want to deliver proper mail and only block SPAM).

Microsoft are one of the worst. You have to search around to find the page where you can report the problem, it is not mentioned in the error text. They ask you which domain you are sending to even though the same error appears for hotmail, outlook, live etc. and then when you submit the request they ALWAYS send an email saying that there aren't any problems. I suspect this is in case you don't have a problem any more and don't take up any more of their time. They ask you to reply if it is still a problem, "yes because if it wasn't a problem I wouldn't have raised a support ticket". You then get some sort of response like "we have set the server to a more suitable level", which I don't understand. Sometimes they say "mitigation is not available" but they are very hard to be drawn into an actual conversation. One person told me that the server was sending erratically and I told him that it would do! Why was this used as a heuristic? No answer. They even sent me a link to a "best practices whitepaper" which was so old it mentioned Windows Vista (I shit you not) and even a defunct protocol called Sender ID, which is now officially dead (largely since 2004 - yes 20 years ago) but still mentioned multiple times. As an aside, what terrible organisation that handles millions of emails per day cannot spare literally a few hours for one of their Technical leads to update a 20 year old paper to actually be useful?

You know who I never have trouble with? GMail. They have a much simpler approach. Check SPF, DKIM and DMARC and if it aligns, put it in the Inbox. If it has been marked as SPAM by someone, eventually put it in everyone's SPAM folder. They can still find it if it's legit.

## Or join the Certified Senders Alliance
These are a Germany-based organisation that provides an independent organisation that effectively vouches for you if you meet a variety of requirements. Most of these are expected and reasonable like "make sure opt-outs are actioned immediately with 1-click", "use DANE for sending" etc. there are a couple of rubs however, an organisation like ours that turns over about £4M per year but makes no money on sending email has to pay $1000 per month for the privilege. Not an enormous amount and they have bills to pay but you pay on your turnover, not on the amount that emails makes for you.

The other problem is that there is no guarantee that a particular provider will honour the CSA registration although it is likely to lead to better deliverability than not being a member. 

In fact Microsoft actually recommend that you join an alternative organisation (Can't remember their name) but this is worse because it is a commercial organisation rather than a not-for-profit and, again, no guarantee it will actually help.

## What about if I only use it for my personal/low-volume emails
You might be absolutely fine but you might fall foul of some clever Dick somewhere who decides that very low volume servers are suspicious and need to be punished.

We seriously considered using a third-party for bulk sending to avoid all of this hassle but SES had very strict sending limits which weren't high enough for us and others we tried are either a bit too religious on blocking things they don't like for every single customer based on one customer doing something "dodgy" or the delay in sending is almost awkward. We tried a company once (I won't name them, they might be different now) and I wanted to use them to send things like account verification emails. When I set up the account and tried it and then waited for 20 minutes without receiving the email and then contacted them, I was told that "new customers are slowed down but will speed up over time". "How can I commit to using your service if I don't know that emails will arrive quickly?" Needless to say, I didn't use them.

So as always, YMMV, run your own mail server if you feel like it, it might be OK but if you would rather waste your time on things you are better at and avoid the grey hairs, pay someone else to do it for you!