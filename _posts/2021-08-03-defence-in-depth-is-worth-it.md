---
layout: post
title: Defence in depth is worth it
date: '2021-08-03T10:36:00.000+01:00'
author: Luke Briner
tags:
---

I was reading [this post](https://news.ycombinator.com/item?id=28043198) earlier about Docker security and read a specific reply which seems sadly too common when dicussing something like security. 
It was something along the lines of "I wouldn't bother since the Linux kernel has a tonne of holes in it".

This really is missing the point. Everyone should know that there are no perfect security controls in IT and only a few really good ones. Firewalls, for example, are massively helpful but you cannot
assume that a firewall will never be compromised and that as long as you have a firewall, you don't need to bother with anything else. 

In the same way, a bank doesn't just rely on having really strong external walls since it needs to allow people inside and requires a large door punched through it. For that reason, a bank has a number
of security controls including guards, locked doors, glass or plastic screens, alarm buttons and a vault. We also know that these controls are not perfect since banks do get robbed but what we do know
is that banks would be robbed much more frequently if they didn't have all of these controls in place.

Many of our apps are like banks. They need to be secure because they contain value in the form of data or in the form of access into a network that might contain other systems of value. Even in a simple
case, a machine might be owned purely for computing power to carry out crypto-mining but we hopefully all know that apps have value in one way or another. We should also know therefore that even though
things like firewalls, user jails, file systems, networks and software controls are not perfect, they are usually all used or hardened by best-practice, by third-party libraries or by humans who understand
how to configure them securely. Fortunately for us, there are many people who have gone before and shared their knowledge about "secure nginx" or "secure php" or whatever.

We need to remember that although the ability to hack does not require wizard skills, to hack a system that has the basic security controls in-place requires a number of things:

1. Time - sometimes considerable
1. Knowledge of your systems/their IP addresses
1. Some knowledge of what hardware/software you are using
1. Sometimes money to pay for automated attacks or marketplace hacking tools
1. The assumption that you are not always evolving security - they wouldn't bother if you were changing things while they were trying to attack your system
1. The motivation to spend all of the time/money/effort on *your* system
1. Often some special knowledge of networking/Linux/firewalls etc. to be able to navigate your system blind

If you don't bother, a script kiddie will hack your system. If you do bother, even if you have a less than perfect control in one place, you will significantly decrease both the number of potential attackers
and also their ability to get a successful hack within an acceptable amount of time/effort.

I would have thought this was obvious but apparently not!