---
layout: post
title: Windows Server VM on Proxmox 9.2 - Arrrgh
date: '2026-08-20T16:18:00.000+00:00'
author: Luke Briner
---

# Windows fails at install time or shortly afterwards with corrupted data
tldr; The processor was defective!

However, the time to bottom this out was long and the AIs weren't particularly helpful, their hardware knowledge pales in comparison to their coding ability.

I was seeing a plethora of error events in the Event Viewer if the installer finished like "Catalog Database: The database page read from the file...failed verification" and "A transient memory corruption was detected" all
logged under ESENT but which breaks the Cryptographic service and therefore also Windows update. It also causes quite high CPU (25%) which is probably various things retrying or fighting with corrupted data. Repair doesn't work
either (at least the way I was told to do it).

If the installer doesn't even finish, either it gets stuck in a reboot loop after finishing file installation or it shows a dialog "The computer restarted unexpectedly..." and tells you to run the installer again.

# Proxmox VE Rules
I like most things about Proxmox:
- I like the freemium model where you pay for support if you need it/can afford it
- I like how responsive it is to manage multiple machines
- I like that it is a Type 1 hypervisor with only a thin Linux Kernel between VMs and the hardware

A few things that annoy though
- You cannot use vxlans for Ceph monitors, which requires some fiddling
- There is a reasonable amount of cli type work to add new nodes and sort out problems
- Some settings are in strange places like zfs pool block sizes are managed centrally even if each node has its own pools
- Windows VMs are a pain in the a***

# Windows VMs
We expect Windows to be a pain.

Firstly, they don't ship with virtio drivers so you have to do another job finding the virtio iso to mount onto the Windows VM and fortunately, Proxmox does this in the GUI nicely. However, the virtio windows ISO is not updated regularly,
I think it is currently about 10 months out of date and the virtio scsi driver has a serious bug that corrupts data (or appears to be corrupted) at high workloads e.g. SQL Server. I have made my own virtio ISO from a later version.

Secondly, the installation is really slow, which is mostly annoying when you are debugging an installation and have to keep running it.

Neither of these are Promox's fault.

However, when it comes to CPU settings and Hard disk settings, details are important and defaults are not to be trusted.

The default motherboard for Windows is x86-64-v2-AES which has some AES extensions but none of the more recent AXD instructions used by a lot of modern software and operating systems. In fact, Clickhouse will error on installation if these are not installed, other things helpfully fallback to slower emulated functions. You can obviously use a newer model but too new and you might not get the same performance on an older host that doesn't support the new features if you ever migrate the VM. Using "host" is also potentially the best for features but if you move it to another host with another process or even Intel vs AMD, you might get unexpected bad results! I usually choose x86-64-v3 which has some of the AXD instructions but not the very newest.

NUMA is a bit hit-and-miss but only comes into play if you are using more than one CPU. The theory is that Proxmox will pin memory down to the CPU cores that you are allocated so that memory access is local to the processor and not "cross processor".

Now Hard Disks are the most complicated and confusing and that is because historically, some features didn't work well with Windows and weren't used but now they do so depending on which article you read, you should definitely choose io_uring for Async IO or you should definitely NOT use it! Using it might or might not have any effect on your workload so maybe you don't notice any issues, maybe it is slightly slower than it could be because everything is flushed instead of cached etc. but I think the best settings for Windows hard disks are Discard=1, IO thread=1, Cache=No cache, SSD emulation=1 (if using SSDs or ZFS pools) Async IO=io_uring. Proxmox will choose VirtIO SCSI single for the controller by default and that is correct. Only virtio gives decent performance compared to any abstracted storage like Qemu. You can optionally pass-through disks but that is extra setup of course and removes the storage abstraction that something like ZFS gives you.

You will need to attach a virtio disk for the Windows installer which won't be able to see the Virtio disks.

If you are using a workload like SQL Server that requires a decent allocation of RAM, you are better disabling the ballooning option on the RAM since it might ask for more RAM but not be able to get it when it needs it.

# When it goes wrong
When I saw all those errors after installing Windows, I had never seem them before and I was offered all kinds of possible issues by the Provider and AI agents (all of which were plausible):
- Enabling Guest Agent before it was installed might be causing some memory blip that causes the host to panic (I did have some RAM issues early on like this)
- Physical memory problems (Ran Memtest86). I wasn't convinced that RAM problems wouldn't surface in lots of other ways. The failure in the event viewer was pretty consistent
- Problems with seating of hardware. Annoying because it took some time for the provider to be able to confirm that everything was reseated.
- Storage issues either hardware or connectors. Ran various stress tests (stress-ng + badblocks) but that all looked OK. Nothing reported by `zpool status` or dmesg logs about memory modules.
- Virtio driver or stack problem. Created a Linux VM with the same hard disk settings as the windows VM and stress tested it. No problems found.
- Various combinations of cache settings on the windows hard disks. Lots of change - reinstall - check - fail.
- ISO integrity issues for the installer. I checked the checksum of the ISO on both this machine and one where it had installed correctly and against the Microsoft list, it was correct.
- Maybe with all the RAM I was giving it (225GB) it was running slightly faster than most VMs that had succeeded so reduced it to 16GB to see if it helped. It did not.
- Tried removing all RAM except one stick and then when it failed, swapping that out for another stick. No dice there either.
- Microcode versions on the kernels (remembering that the other server seemed OK). Both matched and AI confirmed it was an up-to-date version.
- Maybe a bent pin on the processor. Checked it and no problems.

The last possible variable that would have so much effect although seemed unlikely: A broken processor. Broken in a way that makes Windows fairly consistently fail to install but no other noticeable errors (the Windows section?). The provider swapped out the processor for a new one and....bang. Back in business.

I'm not sure what I learned other than a methodical approach changing one thing at a time starting with the easiest things. If AI starts to sound more like Donald Knuth than a helpful IT Support person, it is probably time to pause that line of enquiry!