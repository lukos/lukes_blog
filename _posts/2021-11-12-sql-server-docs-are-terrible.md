---
layout: post
title: SQL Server docs are terrible - setting up an always-on availability group
date: '2021-11-12T15:20:00.000+01:00'
author: Luke Briner
tags: 
  - SQL Server
  - Failover
  - Availability groups
  - TDE
---

I am a very pragmatic man, I know that ideals are targets and not always attainable and I have worked in places where it is quite hard keeping things well
documented. So how is it, that the documentation for SQL Sever is so bad? A colleague suggested it might be an evil plan to push people towards their training
and books but I am not sure. I think, like many companies, MS are so far removed from their customers, that they have demonstrated for many years a terrible
and ever-shifting attitude to their docs.

I really like SQL Server. I think it works well, when it is running it is generally stable and the security model is fantastic. As a pure database engine, I have
only a single reservation and that is the licensing cost is extortionate! At least I only use it at work where someone else pays. Otherwise I don't have a problem
with it.

## Failover groups
A failover group is a simple concept. You have multiple replicas of the same database and if one replica becomes unavailable, you can manually or automatically "fail
it over" to another replica. A great concept, although, again, the licensing model can bite you very quickly, especially if you want any secondaries to be readable!
That aside, you would think that it would be fairly straight-forward to set this up, it is an extremely common pattern for SQL Server and database engines in general.

In fact, I think it is straight-forward enough that someone with an amount of system understanding (me) might not even need to read docs. A good installer would be
perfectly acceptable but no, let us rewind. We actually already have a FG setup in production and want to use an Azure virtual machine as 3rd replica for disaster recovery.
What we didn't want to do was to play around with the live group in order to work out how to do this (and whether it was possible), so I decided to setup a test
FG in some other VMs to match the production setup fairly closely and we could then see if Azure can be setup without needing to do anything drastic. The test was for
this:

* 2 replicas
* 1 Active Directory server/domain controller
* A single encrypted database using TDE

## Installing SQL Server
I had made the mistake of starting with the documents on creating a "domain-independent availability group", which is a new concept needed for Azure where you can mix
nodes that are on a domain (like our production ones) and nodes that are not on a domain, like the Azure VM. This document [here](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/domain-independent-availability-groups) starts very Microsofty so I didn't think that it would be a problem. However, there were links to 2 very poor quality documents
for "Deploy a workgroup cluster" and also "Enable the always on availability groups feature".

The first of these was published in 2015 and has simply been manually modified ever since. It reads like it. There are places where it says things like "must be running Windows Server 2016" even though Server 2019 has been released and is obviously still capable. It then makes glib references to "All servers must have the Failover Clustering feature installed", which makes sense once you know it is an optional server feature, but if not, is this something related to SQL Server?
It then has a mess of random gui/powershell/registry changes that have to be made. This fails for 2 reasons. Firstly, it is simply poor quality linked to from a 1st-class article about a major SQL feature and this feels more like a brain dump.
Secondly, the user should not have to be running powershell, registry changes etc. these should all be part of the SQL installer, or at last an installer specific to failover groups.

The second link points to an enormous document that quickly feels overwhelming, even to the experienced. Asking to change all these different things doesn't sound right. If it doesn't work, you end up with a broken server. If it was handled
by an installer or script, if needed, it could have an uninstaller to reset your server.

I raised a github issue about the docs. It made me quite angry that there must be people in charge of this documentation who don't see how poorly it is written and refactoring it. Splitting it up if the instructions are different for each type
of cluster, not just having loads of "if using this, do this, else that". The fact is, it wasn't actually that difficult to setup once I revisited it from a different angle when I "started again"

## Starting again
After resigning myself to not being able to actually install and configure this and feeling upset and a bit angry (do I need to pay a consultant to run some installers?) my CTO then reminded me that we would need to test this on a separate
system so instead of starting with the "Domain-independent cluster", that I could instead create a standard test cluster, which would remove some variables and hopefully help me to understand which part of the orignal docs I could ignore since
I would already have a cluster setup.

I created 3 VMs, one each for 2 replicas and 1 AD server using Windows Server trial images and SQL Server Development Edition. The SQL Server Installer tries to be helpful but it is very 1990s and simply has about 50 different hyper-links to do different things. Do I choose the normal install? Do I choose one of the various cluster options?

## Some failover concepts
One of the difficulties of learning something new is the jargon. Clusters, availability groups, replicas, SQL Server, nodes etc. what was what?

* Failover cluster - This is the sever/network level system that coordinates the replicas. It is not specific to SQL Server, although this would be its most common usage. This is installed from Windows Server optional components and can be setup before SQL Server is running.
* Node - Failover cluster calls each machine a "node"
* Availability group - A SQL server concept where 1 or more databases can be automatically synchronized across each replica. A listener is used to provide a dynamic endpoint for the database which is automatically changed when the system fails over.
* Replica - Each server that forms part of the failover cluster is a replica when inside SQL Server

## Early problems
When creating many failover groups, especially for SQL Server with expensive licensing, you most likely only have 2 replicas and this means you cannot ever have a majority vote as to which server is available. For this reason, you can add a number of "witnesses" which can provide more votes to help the cluster reach quorum. I tried adding a cloud witness using an Azure Storage account but since I was logged in as a local Admin and not a domain account, the process simply falls over with a generic error about WinRM and Kerberos (neither of which are required knowledge for entry-level DBAs).

Secondly, when installing SQL Server, it is not clear which of the options are needed for "mirroring" or "replication". I just chose a handful that sounded necessary but this is another thing that wouldn't be needed if some slightly more intelligent people actually thought about creating useful installers, a bit like the Visual Studio ones where you can choose a workload, which includes modules to install and which can overlap with other workloads.

Thirdly, it is common to read things like "sort out the firewall", but again, details are left to Google. I found one quite comprehensive list but it didn't even mention port 5022, the default port for AG clustering. I think I got these right.

Don't forget, TCP is not enabled by default on SQL Server. Another item that really should be in the installer. Most people will want to use it and the number of clicks to get it and update it is just another hassle I would rather have not
bothered with.

## Setting up the availability group
Again, the instructions don't even talk about encryption keys but if you have encrypted databases, you have to synchronize any encryption certificates to the secondary machines before even attempting to setup a FG for an encrypted database
(more later). I created a simple database and then started the AG wizard. I then saw an error that I needed to have taken a backup first. Why didn't the wizard just allow me to do that in the wizard? Went out, backed up, back in again. I choose the second replica and it all looked like it worked but it sat there with an error that clearly meant it wasn't working.

No help on the docs again and for some reason the wizard didn't spot the problem but fortunately, in this case, the event viewer did have something: Login failed for TESTDOMAIN\TESTSQL1$. What? At what point did I choose the administrative
use of the machine? I hadn't connected the replicas using that at any point. I can only imagine that because SQL Server was running as a local service, it was able to impoersonate the administrative user when connecting to another machine.
I'm glad I found this because this was really unexpected. I had no idea how to do this properly so I simply added the admin users in both directions to the SQL servers. Then I got another error. This was mentioned in an article I remember seeing and was to do with granting connect to the high availability endpoint. 

After making these changes, everything seemed to wake up automatically and the database finally synchronized. I thought I was out of the woods, but I was not, I had to make another change - an encrypted database!

## Encrypted databases
So we know that Transparent Database Encryption is a fairly useful mechanism to allow copied databases to remain secure and only be readable by someone with the key, but the details of how this works with FGs is more mysterious. I spent AGES working this out but I will spare you the journey.

Firstly, you need to make sure you have a master encryption key on each replica. These do NOT need to be the same or even use the same password. You then need to create a TDE cert on the primary (which will be applied to the database) and then
you need to backup and restore this on the secondar(ies) BEFORE you try and setup the failover replication. You will then enable encryption on the database pointing to your new cert.

You cannot use the wizard to set up the AG for the encrypted database. Why Microsoft? Because they don't have a great attitude to their customers who have to manage these day-in, day-out. Nope. You have to:

1. Add the database to the AG on the primary using script
2. Create a full backup of the source database, copy and restore it to the secondary using NORECOVERY
3. Create a transaction log of the same and restore it using NORECOVERY
4. I then ran a script from my CTO which is supposedly waiting for the database to become available
5. Add the database to the AG on the secondary using a script

Oh my goodness, this was complicated and it took so long to get all the problems out the way. Firstly, I had used some code from google to restore the same master key from primary to secondary (which I didn't need to do) and this had encrypted
the key, which meant it needed explicitly opening every time I wanted to run a command that needed it. This was a recurring issue that created all kinds of random errors. Secondly, I had to make sure the backup files were overwritten so they didn't have the same content in them multiple times. I also had an issue with filenames until I scripted both steps. Sometimes, the destination files were deleted with the database and sometimes they weren't so I had to use WITH REPLACE. Basically, a load of things that wouldn't have been necessary if Microsoft had simply augmented the AG wizard to handle it.

I have now got to a working test system, which has taken about 8 hours with 2 noddy databases. Now that I know what I am doing, I could probably do it in 10 minutes. That difference between the 8 hours and the 10 minutes is Microsoft's fault
because if their docs weren't so bad/fragmented/a mixture of verbose screenshots and shot glib instructions etc. then I could have done this easily. Docs are less important than the software though. Don't make the customer run random PS and registry stuff, get your installer to do it, you know more about doing it properly than we do and its not like you don't make any money on SQL Server.

I fear though that Microsoft consistently fall for the "one new system to rule them all", which ends up creating yet another system. Instead, find things that people ask about on forums and update the docs and then add a link to the docs.

Just please do something.