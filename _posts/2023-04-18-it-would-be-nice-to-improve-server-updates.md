---
layout: post
title: It would be nice to improve server updates
date: '2023-03-18T19:21:00.000+01:00'
author: Luke Briner
tags: 
  - Servers
  - Linux
  - Updates
---

## Anyone else think that server updates are a pain?
There is something that doesn't quite sit right with me. I have, for example, a postfix server doing the same job it is doing now than when I installed it 2 years ago. Yet, for some reason, there have been several hundred OS/library updates since then that have had no obvious effect on the server, except probably slowing it slightly.

I get that there are serious security fixes right? Probably some openssl stuff or maybe some updates to certbot to match changes made by LetsEncrypt or whatever but what about everything else?

* Performance improvements? Don't need them, it runs fast enough.
* New kernel features? Probably don't need them. It works now.
* New driver support? Why? It works on the given hardware it is installed on, the only time this will change is if I move it to another server.
* More library features? Unlikely to need them.

In fact, I suspect that I need no more than 10% of all the updates I have to endure. Now updating Linux isn't terrible but it does involve taking servers offline if they need a reboot and with the 50 servers I manage, it is quite a pain even with unattended-upgrades enabled. I have even seen it where I have spent an hour running `apt dist-upgrade` only to return the next day and find another 10 updates ready to install.

## What's wrong?
I think we all kind of understand the problem right? We have single branch deployments for our relase. If I make performance improvements, fix some rare bugs, improve security, it's all going to go into the same branch so you get all of the updates even if you don't need them. I could accept getting security fixes that I might not need but do I really need performance improvements? New kernel drivers etc?

What I would love to see is a branching strategy that allows a release like postfix-12.0.0 and then after the first release, you can upgrade to e.g. postfix-12.0.1 for all kinds of updates made in that release but you could also track postfix-12.0.1-security to just get security updates and apply those automatically. If you ever want to "catch up" you could then do an upgrade to the full release, perhaps with a switch to `apt` like `--security-only`. This way, I suspect we would have far fewer updates and if we did get updates, we would know they are important because "security" whereas the minute, I can kid myself that most are probably not important and can wait a week or two.

## What else could we do?
I have actually tried Landscape and also SuSe Manager and had hassle with both. I can't remember the exact details but I would have hoped for something like "Log into the server and run this 1-liner" but instead you had to do load of weird stuff with keys and config files, something that seemed very flaky and burdensome to apply to multiple machines.

Other suggestions inclue Puppet/Ansible (really not the right tool for the job imho) and even, "I wrote this script that I run over ssh", which again feels very fragile and doesn't really get around the issue. I think I could cobble something together in Ansible but it just feels a bit easy to screw something up and if you run it against all your servers, which is kind of what it is designed to do, you could break a lot of machines very quickly so I don't really know what the right way is to manage updates on multiple machines.

And Windows Server? Don't even get me started, there isn't enough disk space on this PC to write about how bad Windows Updates are and I have about 4TB!