---
layout: post
title: The SQL Core Licensing problem
date: '2025-08-26T12:38:01.000+01:00'
author: Luke Briner
---

## What is the SQL Server Core Licensing problem?
SQL Server licensing has for a long time been based on the number of physical cores on the host machine, which historically worked out OK I guess. Back when 4 cores was "wow" and 8 cores was "WOWWWW", if you could afford such machines, you could definitely afford to grease Microsoft's palm with a few hundred pounds per month for licensing SQL Server.

However, things have changed now and this can present a problem for those of us who want or need to use SQL Server Enterprise (multiple databases in availability groups and more than 1 standby replica). Today, the price to license 4 cores is approximately $10K per year. Yes, on your server and yes only for 4 cores. You want 8 cores? $20K. So why would anyone want to do this? Quite simply, SQL Server was one of the most reliable database engines before Postgresql was heard of and before MySQL/MariaDB was considered suitable for business (of course, you could always use it if you knew how to debug it/get support whatever) so SQL Server ate the market for people who didn't want to pay "Oracle" money and it got sticky. At SmartSurey, we use SQL Server and like many organisations, have so many stored procs with some complex logic that to convert these to use e.g. Postgresql would be both time-consuming, costly and risky whether you did it with AI or contracted it out to another company.

## Why is it a problem now?
The problem now is that if you want to use SQL Server on only, e.g. 4 cores, and you want a new server, good luck! A bit like you don't get computers with 2GB RAM any more, cores are so cheap that most modern CPUs are at least 8 and usually many more. Your physical server might be a few $1000 but your licensing gets very expensive very quickly.

On cloud, it can be even worse. If you want a lot of RAM for SQL Server to use for cache, how many cores that come with? If you look at Azure's pricing page (AWS and others are similar), a memory optimized Eadsv6 instance with 4 cores only comes with 32GB RAM. If you wanted e.g. 4 cores and 128GB RAM, tough luck! This makes sense though, you are buying a slice of a host VM that probably has like 128 cores and 1TB RAM so you get the relevant percentage of both even if you don't want the cores because "SQL Server Licensing".

Now this page is slightly misleading because actually you *can* get more RAM for your cores if you go into the Azure portal and create a VM (I guess the pricing page might get too big if they had every option in it?). In the portal, you can get 4 cores with 64GB RAM or if you are happy to use a v5 Azure image, even 4 cores with 128GB RAM but still, if you wanted e.g. 4 cores and 256GB RAM, you would have to pay for 8 cores and double your SQL Server Licensing to get the extra RAM!

So it might be possible for the short-term to get a memory-optimised VM on Azure/AWS/wherever and get the correct blend of cores and RAM but if you do then need to go above that, you will need to migrate to the solution before which will entail creating a new instance, migrating data and then switching over.

## Just limit the cores used
In SQL Server, you can actually tell the database to only use up to e.g. 4 cores even on an 8 core machine, and I assume that works OK BUT....the licensing is very clearly based on the physical cores on the host, not on the number you limit it to on SQL Server, which is quite extraordinary but then SQL Server must be an enormous cash cow for Microsoft ("Server products and cloud services" is $98B for 2024. Of that, around a quarter is estimated to be database products).

So what options do you have?

One option is to put more databases on the same host so that instead of e.g. 2 x 4 core licenses on 2 x servers, you could pay for 1 8 core licence for the same amount of money but then you are sharing resources so that might not be the best option for a production workload.

The second most obvious is to run SQL Server as a virtual machine. But, and this is also a requirement, the virtual machine cannot just use e.g. "4 virtual cores", it must be pinned to 4 cores and not be allowed to use time slots on other cores even if it is supposed to add up to "4 virtual cores" because you might get like 1% extra performance and Microsoft would like to charge you for that. This is possible.

## SQL Server as a VM
If you have a bare-metal server, it is easy enough to install [Proxmox](https://www.proxmox.com/en/), VMWare or even, God forbid, Hyper-V and then install SQL Server as a virtual machine as long as you can pin the cores. It is a bit more work than you would like but you get to pin the cpus but potentially pass through almost all of the host RAM to get the blend that you want. The networking and stuff is not too complicated and you solve your problem (although it would be nice if you were just allowed to pin SQL Server to a number of cores for licensing purposes).

On the cloud, it is harder and for a good reason, not for price-gouging. On the cloud, there is already a Hypervisor that you do not have access to (even if you pay for the use of 100% of its resources). This is obvious because the cloud provider needs to be able to control that machine and when it gets destroyed, re-provisioned etc. This means that everything you see and interact with is a virtual machine even if you are using all of the resources of the host, it is still a single VM on a single hypervisor. There is no obvious way around that, if you had access to the Hypervisor, how would Azure take it back once you stop paying for it? Netboot? No thanks.

So you have a different approach using *nested virtualisation*, which is a bit easier said than done because, of course, when you create a new VM, you can't choose "Hyper-V" or "Proxmox" as the image to use, you have choose something like Windows Server or Debian Linux, which is where Proxmox comes in!

## Nested virtualisation on Azure with Proxmox
I will write a separate post about performance concerns and the like but at this point, I will just describe roughly what you need to do. It is not officially supported but it is allowed so don't worry!

Because Proxmox is amazing and is based on the KVM kernel module, it can be installed on top of Debian instead of having to be installed from an ISO-type installer. This limit you somewhat because things like disk setup, which is usually part of the Proxmox installer will need to be done manually before installing Proxmox but there is an [article](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_12_Bookworm) on the Proxmox site that basically describes how to add the relevant packages. It really doesn't take very long and then you open the browser, point it to the Proxmox GUI and you are away.

There is all the usual configuration to open ports, get https certificates and whatever but it just works really. Once you are up and running, you need to create the VM for SQL Server probably running Windows Server (SQL Server for Linux is still a bit hit and miss for me), give it e.g. 4 cores and most of the RAM. You generally won't be running any other VMs so just leave some RAM, maybe 2-4GB for the Proxmox host. You then need to lock down the VM to use 4 specified cores so that you meet the licensing requirements.

The only real pain is that as a nested VM, Azure won't be aware of it so you can't e.g. directly bridge the VM to the Azure vlan. Instead, I just added port forwards for 1433 and 3389 from the host to the guest and treated the host almost entirely like the SQL Server itself for networking purposes. The port forwards add maybe max 0.1 to 1 ms to latency and apparently you can use HAProxy to do it even quicker but until you can run a repeatable performance test on it, that extra latency is probably going to be lost in the noise.

If you have paid for a large host e.g. 16 cores and 256GB RAM and are only using 4 for your SQL Server VM then when you need to expand, you can easily re-configure the host VM to pass more cores to the VM with very little effort and no extra VM rental. Don't forget to buy your extra SQL Server licence though ;-)

