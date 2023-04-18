---
layout: post
title: The ultimate network debugging guide
date: '2021-09-15T13:55:00.000+01:00'
author: Luke Briner
tags: 
  - Business
  - Purchasing
---

OK, so this is a bold title but I feel that I have done so many things over so many years, that I still find myself struggling to get some kind
of networking connection working correctly. If only I had a guide, things to check methodically, something like this!

## The Basics
* The error message is not always helpful. This might be because it was a lazy message that wasn't designed to be helpful but it also happens when
people try to add some "security" by creating unhelpful error messages so as to avoid hackers getting any useful information. For example, "No connection
could be made because the target machine actively refused it", sounds reasonable but you might not be hitting the target machine at all, you might
have a firewall in the way.
* Always try and create the simplest connection and add in each extra step one-at-a-time. It is not always obvious what is causing a problem but if you are
VM inside host, with some firewalls, a NAT router to the internet and something similar at the other end with maybe a microservices cluster, you are going to
find it hard to work out which part is not working. Testing from external IPs, disabling firewalls etc might all help.
* It can help to draw out your network elements so that if you e.g. change the port on something, you remember to also change it somewhere else
* Know your tools: dig, ping, nslookup, tracert etc. are all helpful but you need to understand how they work. Ping hits a specific port using an ECHO request.
That doesn't mean that https should also work because ping does. 
* Not all CLI tools use the hosts file and this can give inconsistent and confusing results.

## Routing
We often think of routing as something that just happens because most of the time it does. When does it not? Often when you have multiple network cards, only
one should have a gateway and the IP addresses should not be overlapping.
* Check that if you have multiple nics, that only one has a gateway defined (usually the one pointing to the internet).
* Check that the nics do not have overlapping IP addresses/subnets, otherwise you might get very sporadic errors
* There are some advanced routing settings you can make in Linux but try and avoid that, since it is just another variable that needs maintaining!
* Check that the IP address you are trying to connect to is not outside of the subnet that you intend it to travel down. This can be confusing with CIDR format
but if your subnet is /24 or 255.255.255.0 then you can only address IP addresses with the same first 3 octets e.g. 192.168.0 and if you attempt address outside
of this, it will usually go to the default gateway which may or may not work.
* Don't forget about the hosts file which might have a hard-coded entry when you expect the DNS to come from a DNS server
* Certain protocols might assume a default port if not specified e.g. https in a browser, smb, ftp etc. do you know which ports it is trying?
* If you are running virtual machines, is the host network setup correctly? e.g. nat or bridge rather than host-only
* Are there duplicate mac addresses on the network?

## Conflicts
* Check that you do not have the same IP address as something else on the network. This is unusual but not impossible for DHCP networks that have run out
of address space and is much more likely with static IP addresses or powering up old/backup servers that might have the same IP as live servers
* If you have chosen your own port for something, it could easily conflict with both an existing service using that port (in which case you would usually
get an error when you start up your software) but it might also conflict with some built-in behaviour e.g. ports below 1024 being privileged on Linux or
having default firewall rules in Windows.

## Firewalls
* It is common to have firewalls on incoming connections but you might also have outgoing connections blocked by default. Check for this.
* Check that you don't have a firewall enabled that you weren't aware of, particularly on an individual machine like ufw, ip tables or Windows firewall.
* Firewalls can block TCP and UDP separately so make sure that your rule matches correctly
* On Windows, each network is Domain, Private or Public and each has their own set of rules. Make sure you know which type of network your request is coming
from to setup the rule correctly.
* Most firewalls blocks will kill the TCP connection and give you a message like "refused" in any decent software. However, some others in an attempt to be
even more "secure" will not kill the connection and instead will simply swallow it until the connection times out.
* On firewalls, as well as web servers, it is usually possible to lock down rules by remote IP address, which is obviously nice and secure, but can you determine
the correct IP to use for the rule? If not, you can often create a rule open to everyone to get something working and then lock it down afterwards.
* If you have locked down to an IP address using CIDR format, make sure that the CIDR is correct, since they can be confusing, especially if you generated it
online.

## Proxies
* If you are using something like Cloudflare proxy for DNS, it only supports certain ports because it is actually decrypting/re-encrypting data and doesn't
want to run this on every single port possible.
* Proxies can have all of their own rules which might be configured separately from your standard firewall
* Threat scanning proxies could block traffic for any of a number of reasons, good or bad, in which case you should simplify your testing as much as possible
to determine that the problem is definitely with the scanner and not something else. You might use something like Wireshark to watch how far the traffic goes.

## NAT
* Network address translation is important but not always easy to visualise. When setting up a firewall, you might not know which of the internal or external
address needs to be specified. In most cases you can enable logging of failed connections on a firewall which will hopefully help you work out what is happening
* If you are accessing a system from a virtual network on a specific machine, depending on the setup of the networks, you might appear as the hosts network
address from the virtual network or it might NAT into the address of the host on the external network. 
* You can use curl or a browser to see your public IP
* Does your wifi or router have time-based access controls for additional security?

## DNS
* Can you check whether the client machine has the correct DNS settings to lookup a target server by name?
* Do you have separate public and private DNS and if it uses different urls, can both bind to the target service?

## TLS
* Does your client have the correct root certificates to verify a TLS connection? If not, you should get some kind of lack-of-trust message but this is not
guaranteed, especially if your client is being called indirectly by another system.
* If your connection requires a self-signed certificate or other client certificate, is it installed? Can the user attempting the connection access the certificate
i.e. it wasn't installed by another user in a private area

## App configuration
* Some apps have different configuration for each environment. Make sure that you both have the correct settings for each environment and that the machine that
has the problem is aware of its environment (for example, does it have ASPNETCORE_ENVIRONMENT defined as an environment setting)
* Some cli apps get their credentials from random places like the registry, files, env variables. Are these set correctly?
* If your cli supports multiple credentials, have you ensured that the wrong ones have not accidentally overwritten the correct ones e.g. if file-based creds have
higher priority than environment variable ones