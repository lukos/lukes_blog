---
layout: post
title: Cloud pains
date: '2023-03-24T16:55:00.000+01:00'
author: Luke Briner
tags: 
  - Cloud
  - Azure
  - Hosting
---

## I believe in the Cloud
Despite the toxic discussions on the interwebs about how Cloud is a scam or a rip-off or something design to hook you in and keep you there, I still believe that for many organisations, it is a perfect fit. Yes, you can buy a server in your office for £1000, install your own Linux and do your own thing, after which you only pay for electricity but that server will last how long? Then you will need to buy another. If your demands are small, a cloud VM could be had for £10-20 per month which means compared to £1000, you could run the server for 100 months or 8 years.

However, you can do much more with the cloud. If you want to add a bit of database, a bit of redis, some online storage etc, these can be provisioned in minutes and although the per-unit cost might be high, at low volumes, it is less than the time it would take you to order and configure your own stuff. To buy 4TB of RAM for your server or a book and a Consultant to make sure you didn't accidentally unlock your database to world+dog so there is a lot to like.

## What is a pain
However, there are three things that have bitten me with the cloud.

1. For any off-site environment, you have virtually no control over it and whether it works. A Cloud Provider might post high SLAs but the truth is, when the air-con broke in Azure London's Data Centre, we had to leave with degraded performance for 6 hours until it was fixed. Nothing we could do, no ability to go into a room, move the server to another office temporarily etc to keep running.

2. The cost cannot always be compared to self-hosted systems. If you host your own SQL Server, for example, you pay a cap-ex, let's say £1000 for your server (maybe another £1000 for a backup), a licence for SQL Server and then that's it, the op-ex cost is relatively low whether you are losing it a lot or not very much. Compare this to the cloud, the smallest "Standard" SQL Server instance might cost you £30/month. If you want a backup server, you pay another £30 (fair) but what if you want a second database? Another £30, unless you change your pricing model or move to a dedicated server or lots of other weird options like using elastic pools to share the overall usage and therefore cost. This cost model is fair if everything is running on shared hardware - if you use it lot, you should pay for more of the hardware/support cost right? Yes, but it can bite you in the ars, as we found out when we realised that to get SQL Server Enterprise edition, needed for replication, it would cost £1000 per instance PER MONTH or we could use SQL Azure which didn't seem to perform consistently unless you start paying the big bucks for Premium level hosting etc.

3. The third is something I have been annoyed by recently and that is that you are limited by lots of niche but sometimes important features that you would totally use on-premises but which aren't supported on Azure. This was driven by the fact that one of our services does not support multiple hosts for its database connection so if we failover the postgres cluster on Azure (running on VMs), this system would break until re-deployed with the updated connection string.

No problem I thought. Just use `keepalived`, a fast proxy with automatic failover based on a healthcheck call. I would call postgres to find out whether the server was a primary and if so, `keepalived` would annouce that it was now the owner of a floating virtual IP address, which could be used to connect to the primary. Perfct. 8 Hours on a PoC and then on Azure? Nothing. Why? Because of the way their network *actually* works which is there is a virtual NAT between every device and the virtual network it sits on and the ARP annoucements from `keepalived` simply won't work on Azure.

OK. What's the solution? How about an Azure internal load balancer? Seems legit except it wouldn't work. No clue why. The healthchecks and monitoring are spread around about 10 pages as is typical for Azure so I couldn't even see a simple readout of "probe = (un)healthy" anywhere so it took a call to Azure for a Support Engineer to tell me that you can't have an internal load balancer have a front-end IP in the same subnet as the backend. Well he kind of said that. He said that you cannot access the front-end IP from the same subnet (although mine was also calling from another peered subnet). So I assumed that maybe because I was peered, the routing wasn't working so that was OK, I would add another network card on another subnet to the same datebase servers and use those IPs for the load balancer backend... (Why doesn't Azure stop you creating a load balancer with an invalid config?)

...well, until I realised that I had stupidly allocated the entire address space with a /16 to a single subnet so I couldn't create another subnet, I had to add another address space and then another subnet. OK, maybe not so bad, that seemed to work OK. I then create a new network card and went to connect it to one of the database servers that isn't normally accessed. Nope, you can't do that with the server running (even though it is all virtual hardware), but I could stop this VM since it was not normally accessed. Added the second card, started it back up and checked that both cards were present and it all looked good. Except ping didn't work.

I could ping to itself (not very useful really) but I couldn't ping it from either another machine on the old subnet or from a peered subnet. I tried telnet. I could telnet on the original IP address but not the new one hmmm. So I went to create a new support ticket and got told that I couldn't raise a technical ticket because I wasn't on a support plan. Interesting since I had raised a technical ticket on Friday but no worries, let me just upgrade to a higher support plan: "An error occured". What error? Some weird ajax error about "Cannot select it when it is already disabled". Presumably a nonsense message but I was not stopped in my tracks, had to go back and create a non-technical ticket about not being able to raise a technical ticket.

I'm still waiting.

So something that we assume "just works" because it's Cloud, doesn't always just work. It is missing features, it allows you to do things that are invalid and then it can leave you without any quick recourse. I have now spent probably 8 wasted hours trying to get this to work and I don't even know if it ever will or whether it will work with postgres.

Obviously the same could have happened on a local setup but at least I would have the freedom to dig around, re-configure etc. to make something work.

Your mileage may vary but although the Cloud seems simple behind it's nice dashboards, it isn't always!