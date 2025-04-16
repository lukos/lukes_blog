---
layout: post
title: Windows Server Deserves to Go!
date: '2025-03-03T09:35:00.000+00:00'
author: Luke Briner
---

## I Really like Windows Server
I have been a Windows user since version 3.1 and I really like Windows from the point of view of its day-to-day operation. However, there are some really nasty monsters that have been lurking for decades that Microsoft has not sort to really update or make user-friendly. In fact, I think it has become such a dumpster fire that I want to leave it behind as quickly as possible, and I will with .Net 8 on Linux.

To be clear, some of these have wasted hours of my life previously but I have recently started trying to setup a basic Dev environment from scratch with Hyper-V, SQL Server, Team City and a couple of Linux VMs. The pain just getting the basics to work has re-ignited my anger at Microsoft for creating something that literally makes them billions of dollars per year but which they don't seem to be interested in improving.

I don't have space to go into detail on everything but I will list some of the things I have encountered so far just on this system and the hurdles I have to jump through.

## Licensing
This is a massive headache for everyone who uses Windows, not because we don't want to pay money to use their OS (obviously we would rather not pay for something if didn't have to!) but because of the complexity. They did simplify it recently but it is still pretty outrageous. You pay per-core for Windows Server with a mninimum 16 cores or 8 core per-processor, whichever is greatest! Of course, you can get Data Center Edition for your hypervisor but that is crazy expensive for non-production workloads and doesn't make financial sense unless you have lots of VMs.

Making this even more complicated is that Cloud Providers are allowed to license their own Windows deployments but you cannot bring your own licences for Windows Server any more. What if the cloud provider doesn't provide additional licences? Tough luck, that's what. Vultr are pretty reasonably priced but you cannot purchase Data Center Edition from them or buy additional licences so their dedicated servers are largely a non-runner for Hyper-V virtualisation.

Oh, and just to rub it in, you cannot licence the evaluation of Windows Standard that you downloaded from Microsoft (and have setup from scratch for SQL Server), you have to delete all of that and download the ISO you get in your MS Partner account and licence the exact same thing.

## Firewall Setup
Why have Microsoft decided to have such esoteric ports open by default ("Cast to device...", "Start", "Cortana") across all interfaces? Surely in this day and age, a very simple question in the installer (or even immediately after install) could ask "Is this machine on a private LAN and you want Windows stuff open by default"? Nope. Not only that, but you either need some weird script or a lot of time to click through various rules to disable them, most of which we don't even know their purpose so who knows if we broke anything in the process? Same with outgoing rules, which I want to disable by default but then you have to play a game to work out what needs opening up again to make Windows work (hint: Almost nothing despite the fact that the firewall is open by default!).

## Support
Microsoft Support consists almost entirely of millions of pages of forums with people who clearly don't really understand what they are talking about offering advice mainly of 3 types: 1) I don't understand the question so I'll answer something else, 2) I'll get you to do something incredibly complex using Powershell/Regedit/Group Policy - a bit like asking someone to fix a problem with their engine noise by re-chipping the ECU! and 3) Reinstall Windows! Unbelieveable, people still say that! Who couldn't possibly understand how long that takes. Also, why?

Even the MS staff show a distinct lack of technical understanding which makes most threads go on for 4 pages and either still end up unanswered or just abandoned by the OP who has probably just given up on life. Here's a thing: I paid £100 for this software and I just want to know what this firewall rule is for: surely there is just an answer to that which all the support team have direct access to. No, they don't. They probably do the same search through 50 years of internet archives like we try.

## Network Setup
On Linux, this is so easy. Literally one file in `/etc/netplan` and you can configure DHCP settings, static settings, name servers and gateways, where needed. Literally zero problems with it. What happens on Windows? It tries to be clever. I can see a domain controller on this network so it must be a domain network (that's fine by the opposite could also be true: the domain controller is not accessible but I need this profile to be Domain so I can use the correct firewall rules to try and work out why it is not accessible). These other networks, they can all be public by default according to Windows, which isn't a bad default from a security perspective (even though the firewall is wide open by default!) but it's the fact that changing it is so difficult.

I have seen a link before that allows you to change the type from Public to Private or back again except I haven't seen it for years. Now, firstly, you have to run powershell CLI to query the profiles and then run another script to update its type. Even this could be simplified by adding a very simple button or whatever to the GUI, why does noone at Microsoft ever do it? I dread to think, they have 100s of 1000s of employees but can't find someone to add a button to a GUI. BUT this is not all! If you reboot, it is likely that the profile would be set back to Public from Private! What? I have just configured this, why would you think it is OK to reset it for me? Because Group Policy!

## Group Policy
The theory of Group Policy, of course, is fine. You can create settings that apply across groups of users or computers, potentially locking them out of things that they shouldn't change. Great. However, the interface is the WORST piece of UI that I have ever seen. Literally, the worst. The organisation of it is confusing, configuring it is confusing, search it is confusing and God help you if you get it wrong, you can lock out an entire organisation! Again, why in all these years could Microsoft have not created a nice new snappy UI for it that organised things where you would expect to find them, that would give you action buttons to "Set Default" or "Create Rule for Group" or whatever.

To solve the reboot problem with the network profile, we have ton configure group policy to set the rule that unidentified networks get set to Private instead of Public. Not really what I want to do because I already told it to be private but Windows still can't "identify" it, whatever that means. Also, I am not running as part of a domain so Group Policy doesn't feel like it is relevant at this point. But it is.

## The Registry
Presumably, this was originally just an internal database that customers won't really need to touch. Clearly this has changed and there are now many things that if you need to resolve them, you have to go into the registry and apply some magic foo.
It wouldn't matter if this was once in a while but so many suggestions to fix things that should just work (like network profiles) suggest changing things in here. Once that happens more than 1000 times, that should be
motivation to expose the setting via the GUI. Honestly. presumably most of the GUI stuff just changes a registry setting so why not just add a toggle somewhere so that people don't have to do here. It is also really slow
and complicated to navigate using the GUI and the people on the forums never provide registry files to run directly - again, another thing they should just have in a knowledge base that they can link
to so we can download and run them.

## Windows Updates
This is ridiculous. I just installed Windows Server in about 2 minutes from ISO but the first Windows update has taken about 30 minutes so far. I think one problem is that it runs
at low priority. Why not have a button when we doing this interactively to say "Run this more quickly, I have jobs to do"? When you have to take down a Hypervisor with all its VMS for maybe
30 to 40 minutes, I need to do it right now, not in the background when it feels like it. I also don't appreciate waiting 10 minutes for "preparing to install" then waiting 30 minutes for
"installing" and then waiting 30 minutes for the reboot to happen and then waiting another 1 hour for the background stuff to beaver away. It's bad. REALLY bad. Even a full distro-update on Linux
only takes like 30 minutes.

## DC Promo - hell no!
I know a fair bit about DOmain Controllers and the various primitives involved in this but could I make this work on a brand new server? Nope I could not. Why not? Well, firstly, 
when you try and promote the domain controller, it performs some tests but doesn't test the network setup properly. So then what happens is you get a red-herring error in The
dcpromo logs that imply that there was a problem with the domain name chosen `Flat name validation of DOMAIN failed with 2136`. Nothing like fake news to fool ChatGPT into giving me fake advice to resolve it. After about 50
different attempts to change things, add things, remove them etc. the eventual issue was that the two network interfaces weren't set in the correct order with the internal (domain)
interface being a higher priority than the external one. That one simple thing that is something I have never changed before is enough to break something like dcpromo.

## Why should it go?
Clearly, Microsoft haven't invested enough in the usability of this OS. I'm sure they will claim credit for the new headless Server Core and probably those other things that people boast about that don't actually impact most customers. The truth is that the issues above are massive hassles for server administrators yet are hardly any different than they were 30 years ago!

Linux solved most of these problems ages ago. No firewall by default. Support is much less needed due to succinct instructions and man pages. Network setup is mostly just a single
config file. No group policy needed, thanks. No registry and updates are blisteringly quick. I do already use Linux where I can but we still have plenty of legacy apps that need
Windows and all of the supporting deployment infrastructure that goes with it.

The truth is that I don't know how Windows could compete with Linux now. Windows Core? Why? Is it really any better/quicker? I still don't see any stats on it. Also, you cannot take
on Linux in Linux's world. The only thing they could do is sort out all the GUI nonsense and the ancient horrible interfaces to their system and then sort out updates so they don't
always have to be backwards compatible with everything else. Maybe, just maybe, they might win me back.