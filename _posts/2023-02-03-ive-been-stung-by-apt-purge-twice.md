---
layout: post
title: What Recruiters don't understand about experience
date: '2023-02-03T16:26:00.000+00:00'
author: Luke Briner
tags: 
  - Linux
  - Ubuntu
  - apt
---

## I've Been Stung by apt-get purge TWICE
I am not what I would call a "native" Linux user. I was taught some basics back in the day and have developed my skills a bit but there are lots of things I don't really know and I have still have to look up the manual to create symlinks in the correct way!

I do really like Linux however for servers and have spent some time building out a postgres cluster across Linux servers. Ubuntu is my distro of choice, partly because that is all I have ever really used and partly because trying to find out which distro might be more suitable is a bit like asking what car I should buy: Far too many opinions that a rookie like me cannot understand.

Anyway, the other week, I was trying to install some packages on Ubuntu. I can't remember exactly what I was installing, perhaps something to do with logging or unattended upgrades. Anyway, I did a cursory web search and found an article which said that the library I needed was installed as part of the package update-notifier. This was a headless server and I was somewhat surprised that it said it needed to install something like 1GB of files but I accepted this because I thought I had no choice.

One install and reboot later and my ssh connections weren't working properly. Fortunately, for some of the machines I upgraded, I had console access via Hyper-V and could open them up directly to find an xwindows login rather than the CLI. I then twigged that I had effectively installed the entire Gnome desktop to get the library I needed when I could have installed update-notifier-common instead which was what it sounded like and only included what I needed. I was a little confused as to why I couldn't ssh into the machine but decided that if I uninstalled the massive desktop install, I could then install the common package and move on. I was wrong.

## Do NOT use --purge
I made the mistake of using `apt remove update-notifier --purge`. This seems correct and most sites you find would advise the same thing. Then I was badly stung. I ran this and it appeared to work, I then rebooted and could not access the machine at all. I could see the login on the console but when I connected in, the network was not accessible so that's why ssh was not working (or anything else connected to it). I had done this to 4 servers via Ansible and this was a big problem!

What I found out was that the `--purge` in its wisdom had removed netplan completely so that on boot, the network wouldn't start. How do you install new packages if your network is down? Well some helpful sites have ways of doing it manually, except `ifconfig` and `ip` are not installed on the minimal server install. Other suggested that you can copy a deb file onto the machine and install it with dpkg, except that requires that you have shared folders since the network is not working. I had to destroy the VMs and install them afresh. Fortunately, 2 of them were created with Ansible so were relatively quick to recreate. I managed to fix one of them somehow I can't remember but the last one, my postgresql barman server was properly dead. This did not have Ansible so I had to build it again from scratch and if anyone has ever done it, even with the instructions, it is very complicated and getting that first set of green lights is pretty worrying.

## Once bitten twice shy?
You would think so but today I created a cluster at home for postgres to test failover in a non-production environment. I had been a bit clever and created a VM in VirtualBox, then installed postgres and repmgr and *then* clone the VM (not sure why I haven't done that before). It's a bit rubbish because of the machine-id, you have to play around with it to make the DHCP work OK but it was definitely less work than running Ubuntu installs and I haven't got vagrant setup either.

It was all great, took me all day to install stuff, document it in Ansible, get it all up and working. Well nearly. I had setup some access rules for the backups and decided instead of restarting services, it had been a while, why not reboot them all. Which I did. Then lost SSH. This happened before with the DHCP/machine-id issues so I waited and then realised the same thing had happened. I could login via the console but no network, no ifconfig, no ip and no guest additions to setup a shared folder.

What had happened? I don't actually know but netplan has gone again. This time, I didn't run an `apt remove` at all but for some reason that I can't ascertain, the network is not accessible on all of the postgres servers. netplan has gone again. Now I don't know what to do except start again and make sure I have the guest additions setup so I can share folders if this happens again.