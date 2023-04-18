---
layout: post
title: Why port SQL Server to PostgreSQL?
date: '2022-10-07T19:21:00.000+01:00'
author: Luke Briner
tags: 
  - database
  - postgres
  - sql server
---

## Porting SQL Server to PostgreSQL
I won't do the obligatory long intro on the fact that SQL Server and PostgreSQL are two of the most popular database engines (but they are!) I just want to concentrate of my first week deciding to port SQL Server databases to Postgres.

## Why? - The Differences
Firstly, you might wonder why I am doing it. Well, all things considered, both databases are mature, field-tested, used by 1000s of companies worldwide and largely have the same functionality. I suspect there is not much difference in performance between the two, that is much more based on using the right constructs for indexes/views etc. more than the engine.

There is, however, one very, very large difference and that is the cost of the licensing!

## The costs on Azure
I am running a bunch of microservices in Azure. They don't get an enormous amount of traffic (1 of the 5 has much more than the others) so initially, we used SQL Azure, which is Azure's Database-as-a-service offering which allows you to very easily/quickly setup a server with, in our case, 5 datbases on it. You don't have to worry about OS updates or service packs on the underlying SQL Server instance, it just works. However, it is not particularly cheap.

Azure have decided that you have to pay either on a per-core basis to get "flexibility, control and transparency of individual resource consumption"!, in other words, you can work out roughly how it will perform because it is priced like a VM. Except, of course, it isn't really. The *cheapest* 2 core 10GB machine is an eye-watering £400/month, especially when you consider that you could get a VM on Azure with the same spec for around £80/month (and on Vultr or Ovh etc. for much less). If you want a replica, it'll cost you another £400/month. Hmm.

Alternatively, you can pay for a DTU-based system. This puts CPU, network and RAM into a kind of blending pot since all DB queries take some of each, and then it charges you for how many DTUs you want capacity for. You have to run some tests to work out how many DTUs you are talking about but on our very low traffic systems, we were easily below the 10 DTUs you could get for a very reasonable £20/month. I say reasonable but that is still the cost of an entire VM on other platforms but hey, in the scheme of things why not.

The DTU model, however, quickly comes round to burn you. As soon as you want to run some big query, for example, you might immediately jump from 2% usage to 100%. What happens then? Connections get throttled and sent the Backoff response and you can lose data unless you are religious with your retry logic. So we found, for our email service, we very quickly had to choose the next tier up - S1 - which doubled the DTUs even though we almost never went above a few percent. So £34/month + another £34/month for the replica.

## The performance on Azure
The penny finally dropped when we started seeing random timeouts inserting rows in the DB despite the DTUs not showing any spikes. In other words, you could be under your limit and still not get acceptable performance. I suspect if I asked MS, they would tell me to pay even more or move to Premium because, why not, but I fail to see how perhaps 200 SQL operations/minute is considered "Premium".

I decided that instead of being ripped off on SQL Azure or switching to vCore and paying £400 per database! Why not just use SQL Server on a VM? We get to automatically share the resources across all 5 databases and most of them barely use the database anyway.

## SQL Server on an Azure VM
Now this definitely wouldn't be worth it on Windows Server because MS are still charging crazy licence money based simply on how sticky Windows is, not because it wins over Linux in any meaningful way. I like Windows and use it every day but it stinks that they think paying $6000 for ONE LICENCE of Windows Server Data Centre, is reasonable. The smallest Essentials Server is $500 and you get none of the good stuff for that.

But hope springs eternal because Azure now offer Linux VMs and you can run SQL Server on Linux! They offer a few flavours and since I know Ubuntu the best, I chose that. However, this is where you can easily be misled. I chose to create a new VM and it offers you whichever type of SQL Server Licence you want. More price-gouging but for Failover Groups, you have to have the Enterprise version. Failover is built into SQL Azure but if you want to do it yourself, you need Enterprise. No worries. I chose Enterprise and the estimated cost was around £200/month on a VM with 4 cores and 16GB, which seemed like a reasonable starting size. Note that although this sounds expensive, I did choose the VM with the high network iops and stuff so for a DB server you do want a premium VM.

I set it all up, started to configure it, it was all pretty easy and then I moved some of the databases from SQL Azure to the new VM, changed over the connection strings on my microservices and proved that it was all good, the DB connection latency was much lower although I hadn't setup any replication yet (even though SQL Azure is supposed to be asynchronous so shouldn't have mattered). I was really happy.

Then my CTO asked me how much the server was. I told him that I thought it was about £200, which was displayed in the VM blade in the Azure portal but I thought I better check. I clicked onto the cost analysis button in the Resource Group and was shocked to see that the VM had already cost £400 in less than a month and the projected cost was a staggering £1,200 per month. Yes, you read that right. A *single* instance of SQL Server Enterprise adds £1000 PER MONTH to an already overpriced VM. I thought the £200 seemed about the right price point *in SQL Server* but no way I was going to pay this. What to do?

I couldn't move back to SQL Azure without facing the same performance/price problems and although we do have an on-premises SQL Server that is already paid for (well, that costs us dearly too!) but as we scale we don't want to put more work on the DB Server, we want to do less work on it.

That's when I considered porting to another engine.

## Should I change and should it be PostgreSQL
We can all easily see the *advantages* of making a change but we need to be cautious. There will not only be differences that we will need to learn but also possibly some unknown unknowns that might sting us later with performance/security/maintenance. We need to think carefully about the change.

In my case, because the databases were relatively small and I already had the SQL Server versions available, it wouldn't be the end of the world if I had to move back for some showstopper reason.

## Question 1 - Should I change from SQL Server?
I think this was quite a strong "absolutely". For small disparate workloads like microservices, a pricing model needs to work with small low-traffic databases not have a minimum price of £20 before you can do anything. To get the performance we needed *now* would have required several hundreds of pounds per month, again, for not very much work. To then scale as we might increase email throughput by 10x, would cost goodness knows how much.

OK, we need to do something, what do we do?

## Question 2 - What should I change to?
It made sense to keep with SQL right? We weren't looking for any "world-scale database scaling" like you get promised by NoSQL databases with all of the code rework that would be required. So what were the options?

Realistically for our small team who pretty much only knew SQL Server, I didn't want to risk any of the slightly less well-known DBs like Cassandra, I think it would either need to be MySQL, MariaDB or PostgreSQL.

I know that both are used in production by lots of companies and I expect there will be zealots on both sides who would say that we should use their particular favourite. Both are open-source, both offer paid support directly or via third-parties, both are free to use. I have even used MySQL before with no real drama.

There was only one thing which pushed me to PostgreSQL and that is what I call fragmentation anxiety. 

You see back in the old days, MySQL was a single thing, it was open source and was invented and then maintained by people who were passionate about what it was. Then the company was sold to Sun who were already teetering on obscurity where it didn't do great and then Sun was bought by Oracle in 2010. Oracle are the opposite of MySQL, they are driven by ROI and not on passion for products. Anyone who has ever used Oracle DB will know how many mortgages you need to pay for it.

In reaction to what might happen after being bought by Oracle, Monty Widenius forked MySQL and launched MariaDB, which shares the same heritage but remained open source although, of course, the development wouldn't stay in-sync any more.

Oracle surely did improve MySQL but now you have a 2-tier model. Sure, you can still get MySQL licensed by Oracle under GPL but if you want all the bells and whistles that they added, a single MySQL EE licence for 4 sockets on a single server will cost you a cool £12K! That there is Oracle prices for something that used to be free!

So what remains is the uncertainty about how good MariaDB is when the main supporters of MySQL are a corporate and the fear that in the fragmented market, it is always possible that the feature you need gets bumped up to Enterprise Edition or the free version stops being maintained and you are in the doo doo. MariaDB might not suffer those fears but is it as well supported if the talent is split between MariaDB and Oracle?

I don't know, PostgreSQL smells better to me. Because the people who maintain it don't directly sell additional services (I'm sure many of the maintainers provide consultancy but it isn't in your face on the website), I don't have the same fears that I will get stitched up in some way or pressured into buying the "Premium version".

Anyway, long story short, I can get Postgres for free on a single VM costing me £200/month on Azure and that will support all of my databases. Replication is included for free (because it is free) so no £1000/month enterprise licence PER server and a much more acceptable cost. The great thing is, with the money we save, we can afford to spend a few thousand on a PostgreSQL Consultant to help us smooth things out and still not be out of pocket.

O Microsoft, how much longer can you carry on gouging?

I will talk about my PostgreSQL experience in the next post.