---
layout: post
title: Claude saved my skin
date: '2026-09-17T10:33:00.000+00:00'
author: Luke Briner
---

# AI is surprisingly controversial in Tech
AI as a Technology tool is incredible. Whether software, design, UX, accessibility, hardware design, electronics, etc. there have been amazing leaps forwards in almost all areas
giving access to many more people than ever for much lower costs. A website that historically took a lot of discussion, design, manual labour and perhaps $5,000 can now be done
in an hour for almost no money just by discussing things with an LLM. Sure, it does create risks for some jobs and that is worrying, but it also creates opportunities to do things
we never got to do before with much lower risk and much less time.

# How it saved my skin
Anyway, I have found that using Claude for hardware/SRE work is a mixed bag. I was building a new environment for the USA and there is lots of manual work that traditionally involved
following instructions from a Wiki, occasionally building Ansible runbooks (whether the effort was worthwhile was not usually clear) and then getting confused if something didn't work
when I did it 12 months after the last environment on newer versions of Ubuntu/Proxmox/Whatever and then reaching for Claude to ask it stuff.

There are reasons why documenting things has not worked well for operations that are not performed frequently.
1. We are often under time pressure and don't want to invest time re-writing all the docs as we go along
1. Things change. Ubuntu 24.04 introduced apparmor which made older things stop working the same way
1. Major versions of software are pinned to major releases of Linux. Installing postgresql in Ubuntu 22.04 (v14) does not do the same thing as it does in 24.04 (v16) or 26.04 (v18)
1. The hardware you are running it on might have changed. We generally use newer hardware as time goes by but we were targetting some older machines that were available temporarily more quickly
1. Every system is always a bit different. Our network on Hetzner is basically locked down by Hetzner and we use vswitches, on our new UK system, we have a single tagged vlan that we run everything across, but
in the new USA system, there is a single physical network that is in access mode for public access and a separate tagged vlan for private traffic. Each of these can be hard to reason about
1. Systems, IP addresses, etc. can also change slightly so maybe there is an extra vxlan or not

Anyway, I was seing a very strange error when my microservice pods were trying to access the postgres database via haproxy. It was sometimes working, and sometimes not, seemingly it would be
fine a few times, maybe for about 20 or 30 seconds and then it started failing. This happened to all the pods across all of the Proxmox hosts and I was sure that this was exactly the same setup
I had used in the EU that worked fine (hint: it was not the same!).

Although I am embracing the automatic mode for coding with Claude, due to my previous experiences, I am more likely to use chat mode for hardware: "I have setup microservices on these vlans, talking
to postgresql via haproxy and pgbouncer...." and letting Claude give me hints. I find this is not effective and I think the main reason being that the context gets eaten up quickly with the large
debugging outputs from `tcpdump` or whatever and the focus starts to drift. This is a well-known problem but it is hard to manage since clearing the context regularly can also lose a lot of
what was already tried. I spent about 3 or 4 hours trying to work out how on earth this was happening, Claude asking various questions, which I already knew the answers to, assuming things it
could have just asked me instead of wasting time and tokens and I got to the point where I had no more ideas why it wasn't working. The answer must have been staring me in the face but despite
all my comparisons with known-good systems, I couldn't spot it.

We were under so much time pressure. We had a customer who wanted to on-board and the system wasn't even running. I had an idea that I hadn't thought of before as a kind of Hail Mary but it actually
made sense. I created a basic markdown document describing the archiecture of the new system (about 3 paragraphs) and then another doc with the connection details to all of the proxmox servers (4 of them)
and credentials for OPNSense which was running as a VM. I then set Claude to use Opus on the xhigh setting and told it my problem. After checking it could access all of the server I just left it. It was
amazing. Instead of it telling me to run various things and the usual to-and-fro because the command is not quite right etc. it ran its own tests, capture its own pcap files and even told me "I have left a 30 minute
pcap running, can you send more database traffic". After not very long, maybe 15 minutes, it had found the problem!

# The problem
It was obvious once it found it! My microservices were on a single subnet `10.22.8.0/24` and had to connect to the haproxy database connection which was on `10.22.6.0/24` and therefore went via its
OPNSense gateway on `10.22.8.1`. The connection reached haproxy and it sent back its `SYN ACK`. However, the haproxy server had a connection on the `10.22.8.0/24` subnet so it could send that message back directly
and not via OPNSense. When the microservice got the `SYN ACK`, it then sent back its `ACK`, which again, would have gone via OPNSense. However, OPNSense is stateful and would have thought, "why do I have an ACK when
I have only seen SYN so far? BLOCK".

Why did it sometimes work? Depending on how the connection was established and whether the initial message was small enough that the client didn't realise there was a problem, it could have reused an existing connection
without needing the TCP handshake, and that would have worked. When things timed out, it would be back into the problems again.

What were my other options? There aren't many people in our org who have the same low-level network understanding as me so it would have been some kind of guessing session while under time pressure and while
everyone is busy with other stuff. I could have found a Contractor but that would have taken more time and there is no guarantee they would have found the problem in a reasonable time.

Thanks Claude! I know that not everyone likes you but you saved my skin.