---
layout: post
title: The hidden danger of refactoring
date: '2021-11-01T10:30:00.000+01:00'
author: Luke Briner
tags: 
  - .Net
  - Azure key vault
  - Containers
---

I like to think I am a seasoned problem-solver and fault-finder but I wasted an entire day debugging a Kubernetes deployment that wasn't actually broken
caused by a combination of a sloppy copy-and-paste and several concurrent differences that made me go down the wrong path.

## The Problem
We use Azure KeyVault to store various things like dkim keys and psks for microservices and they work fine in production called from apps running under IIS
on Windows Server. We also have a new PoC app running dotnet core inside a Docker container on Kubernetes. Unfortunately, like a lot of PoCs, it gets put
down and picked up a lot so it isn't always easy to remember the last time it worked and what might have changed in shared libraries since then.

I was replacing a DR microservices cluster and noticed that the PoC app now errored. Of course, my first assumption was that I had setup the new cluster wrongly.
I had, I needed core DNS to get "simple DNS" working in Kubernetes instead of configuring calico or whatever and after several hours of messing/trying different
things and installing a load of Linux tools into the image so I could debug, I worked out that DNS was working, calling out to the internet from the container
worked but for some reason, this call to the Azure key vault kept failing.

## The debugging
One of the things I intensely dislike is unhelpful error messages that try and hide important facts in the name of security. Even with logging enabled, the best
I could see was "Bad Request" and it appeared to be coming from the call to the key vault. This of course started a string of bad assumptions, "it works elsewhere",
"I can call to the internet from the container OK", "I have logged the creds to make sure I am definitely passing them to configuration correctly" but alas,
nothing. I could only assume that some weird container/Kubernetes/Microk8s thing was going on (fortunately, the DR site is less complex than production so I could
remove other possibilities like threat scanners etc.).

I also thought that I had run it successfully locally, which was even more confusing. I came in this morning wondering what on earth I had left to try. I ran it up
locally again and now I was getting the same error locally! (what the... and also, thank goodness). This removed the container variable since I was running it
directly in IIS Express. It also allowed me to debug into the code and find out that it was, indeed, a call to the Azure Key Vault that was failing. I knew it worked
in another app and I knew I had the correct creds so I took a look back at the code and that's when the bomb dropped! 

## The solution
I had actually made a library change earlier "while I was under the hood" to refactor the Azure key vault client from the old to the new nuget packages and that involved 
a slightly different constrction of the client itself but which we had already done elsewhere so I copied the code over to create the client, build everything and only
had to make one other change (or so I thought!) the response from GetSecretAsync is now `Response<KeyVaultSecret>` which ridiculously has a property called `Value` or type
`KeyVaultSecret` which itself has a propertry called `Value` so you get code like `Result.Value.Value` which seems sloppy - it could have been `SecretValue` anyway...

The issue was that the client methods have changed. They don't take the key vault URL any longer because that is built into the client (e.g. one client per key vault). BUT
the method does still have a two string parameter method, instead of `GetSecretAsync(url, secret)` it is `GetSecretAsync(secret, version)`. Yes, that's right, MS refactored
their code but have matching overloads so as soon as you change the client to the correct type, all the compilers errors disappear but will, by definition, all be wrong.

Now yes, I could have been more careful and I did notice that the other code that was already changed did indeed have the first parameter removed but also making such a big
change without making it noticeably break I think is an unfortunate pattern, especially in the absence of Analyzers or specific error messages like "First parameter cannot be
a URL see https://etc".

The bigger picture is still relying on organisations, however large and well-paid, to set the gold-standard for development but we are still like law and medicine were hundreds
of years ago when anyone could do it if they wanted, regardless of how much of a hack they were. Documentation is often poor (most key vault documentation have no examples),
and those of us who don't benefit from working in an organisation with supposedly "world-beating" developers, don't have anything to allow us to learn the right way to do it
so that we don't cause some other poor sod to waste a day fixing something that is so simple, it belies belief.