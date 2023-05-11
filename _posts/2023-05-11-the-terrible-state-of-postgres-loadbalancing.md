---
layout: post
title: The terrible state of postgres load balancing
date: '2023-05-11T11:14:00.000+01:00'
author: Luke Briner
tags: 
  - Postgresql
  - Postgres
  - HA
  - Load balancing
---

## Postgresql is really good!
Truly. I have had no real problems using Postgresql since my previous experience has been (some) MySQL and mainly SQL Server, the change was largely painless. Sure, you have to learn some SQL changes like the differences with quoting, with naming, with procs and functions and with the general performance differences between different types of queries/indexes but fortunately, I was able to learn these relatively easily.

## Azure and SQL server is expensive
In fact, the main reason I used Postgresql was that we needed a high-availability replicated system hosted on the cloud for a relatively small database (well, 5 databases but most are very small, only 1 is about 4GB). Quite simply, you cannot do this for cheap with SQL Server. SQL Azure pricing doesn't scale well and is astronomically expensive in very large sizes. SQL Server VM requires the Enterprise edition which is, wait for it, $1000 per node per month, even if you only need it for a small DB. Also, we had very erratic performance with SQL Azure, even though we weren't close to reaching its Service Limits.

## The high availability struggle
And then it all starts to go wrong. I think Postgresql struggles with a bit of a dual-personality thing. The main engine and a few of the tools are professionally maintained and are relatively easy to use and setup. Following advice in https://amzn.eu/d/jdSoyFa, which itself was quite inconsistent, I setup barman and repmgr.

## Barman and repmgr are also good!
Barman (Backup and Recovery Manager) is an open source project which provides some high-level wrappers around postgres functions to setup regular backups of the main server. These can (and should be) on a separate sever so that backups are kept safe from any primary server failures. In fact, eventually I decided to backup both the primary (for 10 days) and 1 of the replicas (for 5 days) since I realised it was very easy for a backup to be broken by other changes and I wanted a backup for the backup! If you haven't already seen this: https://youtu.be/tLdRBsuvVKc watch it and then check your backups!

## Barman
Like several things in the Postgresql world, there are some options which are not really described very clearly (the barman docs are pretty badly formatted), about streaming vs file sync. To be fair, some of these are related to older versions of postgres but trying to understand how you should choose either method (or perhaps both) did take me a while. Anyway, I managed to get the backups running and it looked fine.

## Repmgr
Repmgr is another open-source app which also provides high-level wrappers around postgres commands to do a number of standard operations including setting up replicas and also controlling failover scenarios. It tracks this with a repmgr database that runs on each server. Repmgr also, importantly, can pull backups directly from barman when creating new replicas so avoiding the need to copy an entire (online) primary server each time you need a new replica.

Again, some trouble understanding why things did and didn't always work, probably mainly lacking the documentation to clearly articulate why some things did or didn't work. A big issue here is the need for repmgr on each node to be able to access every other node with *sudo* powers so that it can stop/restart other servers during failover. You really have to understand which users are running as whom and need to create a password-less sudo config, otherwise repmgr simply sits there waiting for a server to shutdown which won't and them times out with no helpful error.

Again, I eventually got this working and documented everything so that now failover can take place fairly easily (I didn't configure auto-failover since failover should not normally be treated so lightly since it can easily cause problems, especially if the auto failover doesn't work and causes a split brain!)

## Why not do a tutorial?
Honestly, I have sat down a number of times to create some YouTube videos on everything I have remembered but each time I go through it, I find something else that doesn't quite work as expected or that I don't quite understand and I want the tutorials to explain clearly all of the things I didn't know before!

## Then we look at load balancing
There is a really cool feature of the npqsl library for client connections where you can put multiple hosts and get it to try them in order (or randomnly) to find e.g. one of the read-only replicas, you can do the same for the primary connection and add an "Intent" to the connection string to do this. If you pass the intent, it will query each host for primary or replica and you can e.g. require the primary; prefer the primary etc. for auto selection.

### Multiple hosts don't work exactly as wanted
This mechanism is only partially helpful though. In *normal* usage where all servers are running, it works transparently (albeit theoretically with requests to some inappropriate servers before finding the correct one) but it doesn't work well for any failure scenario for a very simple reason. Regardless of whether you are using "in order" or "random", the same basic issue exists if a server goes down.

Imagine our hosts are pg1, pg2, pg3 and imagine pg1 is the primary and this is the primary connection string and we are using the hosts in-order. Great when things are working: We go to pg1, ask if it is primary, it says, "yes" and the client uses the connection. Cool...

But...imagine we failover from pg1 to pg2 now. The next attempt to access pg1 will still be made but it will time out, then the client will choose pg2 and it will work but with every subsequent connection taking "timeout seconds" longer. Every single one! This requires you to use a retry mechanism in the client to ensure the apps don't error but clearly it is no use for any longer than necessary to update the connection string to include e.g. pg2,pg1,pg3 instead. If you fail back, you have to do it all again. Hmmm.

### What are my options?
I have tried various things that are not specialist. For example, DNS is already a way of redirecting traffic based on hostname => IP so that should be OK right? Well, with our Kubernetes microservices and our postgres VMs running in Azure, there wasn't a solution that seemed to work which didn't either add latency to every request (lots of DNS lookups) or which provided a relatively fast failover of DNS entries to point the services to the new primary.

Then it gets really confusing because it seems like a lot of people have solved one or two of the following issues but no-one has solved all of them:

* Connection pooling to reduce the amount of https work and reduce the total number of connections in very large environments
* Ability to load balance across multiple replicas
* Ability to track primary and replicas in the load balanced sets after failover
* Possibly a virtual IP to avoid needing separate load balancer servers

Now, you would imagine that something like this exists, some options include:

* Citus
* pgpool
* pgpool2
* pgbouncer
* pgcat
* haproxy
* keepalived
* Patroni
* pg_auto_failover
* Cloud options
* Envoy

But...and it's bad, all of these suffer from some kind of problem.

* Some appear to be either unmaintained or in some cases, don't appear to have ever supported "production use"
* Patroni is very much geared around cluster management rather than load balancing
* pgcat won't follow the primary after a failover, you need to update its config some other way
* Cloud options are sometimes costly and only work on certain cloud providers like GCP
* Some of the generic options like haproxy and Envoy are sometimes much more complicated due to the fact they are not designed for postgres specifically. For example, haproxy has a postgres health check but it only checks the database is running. If you want to differentiate primary from secondary, you have to use a complicated TCP binary check that effectively encodes the call to in_recovery().
* Things like pgbouncer were designed for pooling so they don't include a lot of other functionality like load balancing and healthchecks
* keepalived would be a reasonable option but it is quite hard to understand its configuration and it uses VRRP to pass around virtual IP addresses. Fine except it doesn't work on Azure so that is garbage.
* Citus is a whole solution, not just load balancing and failover so you would have to maintain a lot more than you might want just to get the functionality you need (I assume their service does this)
* Some of these are low-level commands that do not offer you much by way of automation of the various moving parts

So if you Google this stuff, you end up with a plethora of complex, multi-service or specialist articles about something that should be as easy as "Install, configure, deploy". The worst bit is that many of these articles, in a nod to the understood complexity of what the proposed solutions involve, end with something like "There are many other solutions available [if this one sounds aboslutely horrible]". The only thing is there aren't! Just other equally confusing, undocumented, non-feature-rich solutions that might not be claiming to do everything but which, in many cases, mean they are of limited use.

## Do it yourself then!
Well, It is hard to complain about something you didn't pay for (although we did pay a large sum to a consultant to help us setup postgres since we were saving money on the database engine) but I guess I am not so much complaining as saying that this seems an unusual problem to have not solved and is creating a lot of overhead for people who might otherwise want to use Postgres in production. I am particularly surprised that someone like EDB have not added a load balancer to their arsenal or useful products like barman and repmgr. I also find it amazing that a database that is so popular doesn't seem to have a standard way to do this that doesn't involve writing your own services.

I can only assume that Apple, Instagram, Spotify and the rest simply have big expert teams who can afford to configure these various things for whatever they are being used for, although it would be nice if they could also release them to the community so the rest of us don't have to waste 2 months doing something that seems too complicated.

For now, I am just relying on the Azure VMs being high availability and doing manual failovers very occasionally during the night when traffic is low.