---
layout: post
title: Being careful with AI and ChatGPT
date: '2026-01-12T20:03:01.000+00:00'
author: Luke Briner
---

Happy New Year everyone!

## AI is definitely here to stay
Let's get something out of the way, AI is not going anywhere! For the people who are still complaining about "hallucinations" and what it is doing to jobs, I'm sorry but you just need to get over it, it is too useful and like the wheel, the engine and electricity, the worms don't go back into the jar.

That said we need to understand better how to use AI well for coding/infrastructure assitance otherwise we might go to an early grave because there are definitely some major frustrations for me and I found this out recently with a foray into Proxmox + OPNSense + Ceph + Talos Kubernetes + Hetzner!

Yes: lots of new things to learn and lots of knobs to twiddle but it should have been simple enough and who better to turn to than my old friend ChatGPT?

I will use the term AI when I mean ChatGPT specifically, "just because"

## RTFM
The truth is that most documentation for software/systems etc. is dreadful. The good stuff is usually just about good *enough* but some of it is patchy; some of it out-of-date; some definitely leaves out a lot of the important details (e.g. it has a Quickstart and nothing much else) and as soon as you hit an unexpected hurdle, you are immediately screwed unless people on the forums are helpful. The fact is, I feel bad asking what seems to be a noob question that has probably been asked many times before but no-one on the project is keen enough on the docs to make sure that all "stupid questions" get rolled into the docs.

## Back to ChatGPT
So instead of trusting that the docs have my back, I suspect like a lot of people, I turn to ChatGPT to summarise best-practice and most importantly: step-by-step instructions.

Like a lot of you, I understand a fair amount about hardware and software, about network layers, about virtual networking, firewalls and whatever else but that doesn't mean I want to just stand up 3 Proxmox Nodes and try and "just get it to work", I want to make sure that I get it production ready.

YouTube videos are *almost* always produced by hobbyists who say things like "these are just virtual servers" or "I will only use one node but you might use more", in other words, they are not useful for I actually need in production. How to make sure I don't get locked out by the firewall, how to make sure I use the right settings for my VMs and whatever so back to ChatGPT.

## Problem 1: You don't have enough context
When you ask ChatGPT questions, there is a hidden System Prompt which is added to your question and which says things like, "You are a helpful AI answering questions...you must not be rude...you must answer confidently in a kind tone...", which all forms the context. You might wonder why this isn't just hard-coded into the model and the answer is probably that it would be too complicated and would make your model a bit limited. Anyway, when you ask something like "What do I need to know about production Proxmox clusters?" you might or might not notice that you are not giving enough context. If you asked a human that question, they are likely to ask, "What do you mean specifically?" but ChatGPT doesn't do that because it doesn't understand the context, it doesn't understand Proxmox or anything else, it just infers the most statistically likely response and unless you give it almost nothing, it will happily churn out a response that might or might not be very helpful.

Of course, if you know it isn't helpful, you are likely to come back with something more specific: "I meant specifically how many nodes I need and are there any default settings that need changing for production".

The catch is that when you don't realise that you haven't given enough context and don't know if the answer is good or not, you fall down the rabbit hole and can end up borking your cluster (yes I did and had to start again!).

## Problem 2: AI doesn't distinguish between versions very well
Unless you explicitly tell it the versions of software you are using and sometimes even if you do, the answer you get back might not be correct. Again, if you notice it, you can reply that it doesn't sound right and it might work out that you are on another version but again, if you don't notice, it might be confusing at best and at worst lead you down the rabbit hole.

The thinking mode is better, of course, since it will search the web if it knows that the score of a response is low, but it does, of course, rely on online documentation to build a sensible response and if the docs are not great, the response from AI might also be...not great. It's horrible being told something isn't possible, attempting some gnarly workarounds only to find out later it is possible so be careful with that and tell it what versions you are using.

## Problem 3: It doesn't remember everything in the conversation
If you are talking to a person about Proxmox and you carry on for a while without mentioning it, the person will remember that you are still talking about Proxmox. ChatGPT doesn't because it only has a certain sized context window, which although big, is not massive and for some of these long conversations, it can lose important context and then your answer about e.g. "creating a VM" becomes generic and not Proxmox-specific.

I try and get in the habit of starting new conversations regularly and starting from a clean prompt to summarise where I am since most of the previous context will become irrelevant.

## Problem 4: It can't tell the difference between "easy" and "correct"
I was asking how to ssh into an OPNSense VM since it didn't seem to work. ChatGPT sent me down a hole about enabling and starting the sshd service (which didn't work) but each rebuttal digs a deeper hole with more esoteric and potentially harmful instructions like adding symlinks, finding the executable, creating a service file etc. all of which *might* have been correct, but fortunately, I spotted the seemingly over-complicated answer, did a Google search instead and found out there is an option in the GUI to enable SSH. Dead quick and simple but for whatever reason, ChatGPT didn't know.

Another scenario that ended up ruining my Ceph cluster and making me fear that I was going to need to install Proxmox involved over an hour of networking nonsense when the underlying issue was that a vLan had the wrong IP in it and the firewall didn't include a new subnet I was using. An experience engineer would have asked some better questions about "did it ever work" etc. and then is likely to have said, "try the firewall first". ChatGPT started coaching me on multicast floods and route tables, which was quite interesting but it felt like it should have been easier to get to the truth more quickly.

## What I would prefer instead
I would LOVE it if people who maintain open-source software created AI powered help documentation. Rook io for example, has a load of helper scripts but no obvious route from the home page to, "This is exactly what you need to do to setup a rook cluster with external Ceph". This was another thing that AI made into lots of manual steps, when all I needed was to run export-cluster.py and import-cluster.py pretty much! The Google search AI was much better and gave me about 5 instructions compared to the 30 that I was given by ChatGPT (and which didn't work anyway)

Lastly, a big shout-out to the amazing people who build and maintain tools like ss, ip, grep and others, which are so powerful for debugging (once you learn the magic switches!) and mean that, at least on Linux, debugging things like ARP and route tables is childs play!

So more on the Proxmox stuff later. I am still trying to get OPNSense HA working (it semi-works :-) then I might knock up some more Production-quality videos about setting everything up and the pains I endured in the process!
