---
layout: post
title: Look after the small things
date: '2021-09-10T17:04:00.000+01:00'
author: Luke Briner
tags: 
  - UX
---

Twice today, I have been happy because of small things that someone took the time to do that made a big difference to how long
I needed to spent debugging.

The first was trying to build a .Net app in a Windows container. Why would I do that? Because Linux only supports dotnet core
and I am not prepared to invest the time to port an MVC 5 app to dotnet core. Hell no! Anyway, I was getting the dreaded blank
screen of death (but a 200 response from the server) and I couldn't work out what was wrong. I eventually realised it was something 
related to this weird Javascript engine thing that we use for our React listings.

Now if I had just seen an error like "cannot load Javascript engine", it wouldn't have helped me much because I have never really
developed this part of the app and besides, it works in Visual Studio, just not in the docker container. However, the kind people who built the library
added a little extra information to the error message, "You also need to make sure you have installed VC++ redistributable 2017". Wow,
so obvious a hint but so rare in the libraries I use. Most of the time you get a blank call stack and you might get some half-hearted
message like "couldn't load engine" but someone made my life a little bit easier.

The second was half-helpful but while installing the clair scanner from quay.io, I pointed it to the database server and when I tried
to run it, the logs said words to the effect of "you need to install uuid-ossp". Now I am very new to postgres and I didn't really know
what that meant but the builders of clair had decided to not just log "failed migration" or "cannot generate guid" or something half-hearted,
they were specifically checking for something that is not installed by default and which is needed for their functionality to save some
people (like me) a lot of time.

These examples are for development but the same thing goes for whatever you do. Most of your customers will use a very small percentage of
you app/shop/web site so make sure the high-traffic areas are very slick. Also, new people might not know how things work so a small change
that considers new customers will really help, is it clear in your e.g. cafe whether people need to order at the bar or from their table? 
How many of you have gone into a timber yard and have no idea, do I pay first? What if I need to look before I order? etc. I have heard many
times from people who won't use a shop/website/service for the basics.

It is easy to get seduced by the appeal of a "big feature" but if you neglect those small things then you will be losing customers. Is your
shop drab and unloved? Is there enough space to move around? Do people answer your phones quickly and politely? Does your web site have such a great
journey that customers don't need much support? Get these basics wrong and bye-bye instead of buy-buy!