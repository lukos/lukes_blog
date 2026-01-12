---
layout: post
title: OPNSense error The backup firewall is not accessible (check user credentials)
date: '2026-01-12T20:46:01.000+00:00'
author: Luke Briner
---

## Setting up OPNSense HA
I watched a video and it "just worked" but on my system, I just kept getting "The backup firewall is not accessible (check user credentials)" from the primary end.

Obviously, if you have the firewall locked down, you need to make sure that the firewall has ALLOW rules for things like pfsense and http/s but if you like, you can allow everything to make it easier.

Still no dice!

The problem eventually turned out to be that the MTU setting for the interfaces was NOT picked up from the virtual NICs attached to it so any large packets e.g. https handshake were being dropped by the network causing the misleading error message.

All I had to do was go into the interfaces and set the MTU manually but it was annoying that this was another thing that isn't documented anywhere because I guess it isn't OPNSense's problem and to be fair to them, the best they could do is say, "don't forget to check MTU if you are on a vLan" but anyway....