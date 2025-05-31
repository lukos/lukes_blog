---
layout: post
title: Microsoft need to sort out their email
date: '2025-05-31T12:23:00.000+01:00'
author: Luke Briner
---

## Sending Email Should be Easy
I know people often say that running your own mail servers is hard, but actually it is fairly easy once you understand the basics. However, even when you do this, email providers can still take unilateral action against your sends and you have no visibility of what and why and very often nothing you can do about it. The very worst offender is Microsoft!

## Setting up your server
We use Postfix to run our mail servers. Open source and with all of the switches and dials you need, it is a great system and uses very little resources on our Ubuntu Linux VMs. We currently have 14 mail servers with static IP addresses at SmartSurvey. They run at a number of hosting providers so that for e.g. provider maintenance, we can easily disable maybe 5 or 6 of them and have enough to still deal with the 100K or so emails we send every day on behalf of our customers.

We have the servers configured automatically using Ansible for consistency and I really like [MXToolbox](https://mxtoolbox.com/diagnostic.aspx) for testing them when they are ready to deploy. It looks for the correct rDNS settings, that the server is reachable and isn't an open relay. All a good start and even though the script will run the same on all new servers, I always use this as a starting point to make sure nothing is missed. A few times, Vultr has managed to lose the rDNS setting, which some providers will reject mail over so it is always worth the check.

Then comes DKIM and SPF. All very straight-forward and if our customers want to use their own domain to send from, they can setup DKIM records that allow us to sign their emails on their behalf. Despite all of this, we still have problems sending emails and it happens at completely random times.

## Having a healthy sending culture
When you send on behalf of other people, it gets complicated because you can have Anti-SPAM Policies and Terms and Conditions, but the simple truth is that a lot of people either don't understand or they don't care. They do stuff like take email addresses off of paper forms and instead of verifying those addresses, they simply upload them to us and send out invitations to surveys like "How did we do?" (Usually a bad idea but that is for another blog post).

We now automate a lot more of the checking and will be introducing more explicit confirmations when people upload email addresses that they have to verify, "I have consent to use these addresses", "these addresses have all been externally verified as being correct", that kind of thing.

## Problems sending email
I know when we get problems because the deferred queue for the mail server starts spiking. We will often have a handful of emails that could not be delivered because of incorrect email addresses, full mailboxes, etc, but spread across all the mail servers, it is not usually more than 5 or 6. After 4 hours, these are all bounced and the information sent back to our systems so that subsequent sends to those addresses are blocked before they are attempted.

If the number goes above 10 for 10 minutes, the graph on my dashboard goes red (which I only have on when I am working) and also I am sent an email by Grafana, which I will usually take a look into if I am at home. Invariably this happens to random servers at random times and is almost always Microsoft blocking a server. The receiving system sends back an error message, something like, "The mail server [1.2.3.4] has been temporarily rate limited due to IP reputation. For e-mail delivery information, see https://postmaster.live.com".

This link takes you to one of the cheapest looking websites with a basic text dump of the sort of thing I could probably have written off the top of my head and which doesn't really help. I run my own mail servers, I don't need to be asked "Are you sending email from new IPs?". The error code from my message is not even listed in the table at the bottom of the article.

This makes me angry. Microsoft process millions of emails per day and is a dominant position to affect who does or doesn't succeed sending emails. Their business turns over nearly $250 billion per year. To have a help guide that isn't complete and doesn't even have any structure to it, which would literally take a decent new starter about a day to sort out is terrible. How about separate pages for "My server has never been blocked" and another for "My server was fine before and is now blocked"? How about even just plugging into your mail system and telling me why my server is blocked.

## Feedback reporting systems
Microsoft like a few other providers has a system that you can register with, prove the ownership of your domain and keep an eye on statistics for your mail servers. If you are sending a lot of content marked as SPAM but don't know about it, you can see on their site that the server is red, in which case you might disable it for a month or even tear it down, get a new IP address and start again. That would be fine except when my servers are being blocked, they are not marked as red, they are always yellow (never green), despite the numbers showing that the amount of SPAM is much less than 1%, which should be a green. This is another very poor quality site, possibly from 20 years ago that no-one maintains. Why not? Why not take your role seriously?

You know what Fastmail and GMail do with emails they think are SPAM? They put them in the SPAM folder and never block our servers. That is what you should do because genuine emails are lost by Microsoft blocking a server that has no way to retry because the block last for several hours, even though the message usually says, "has been temporarily rate-limited". I can live with rate-limited because the numbers that are blocked before I notice are usually quite small, like 20 to 30. I would be happy for those to be allowed through at 1 per-minute but no.

## Reporing a problem
Microsoft do have an Outlook support page where you can report a server blocked. They don't ask for a lot of information, but I feel like it is another site that is more or less a "Google Form" and doesn't allow, for example, to sign-in and it remember your name, email address, company web site etc. Anyway, this is the process:

1. You fill in the form and press submit. It tells you a support ticket was raised.
1. Sometimes, the form doesn't work so you are screwed because companies don't want to be contacted so there is no e.g. "If you have problems with this form, please email outlookformproblems@microsoft.com"
1. You receive a reply, usually within 15 to 20 minutes, it ALWAYS says, "We cannot see any problems with the server listed. If there is still a problem, reply to this email with the following data <all the data you already entered into the form>"
1. You have to reply, *every time*, telling them that funnily enough, you don't report problems that don't exist because you are busy and have more important things to do and all the information they are asking for is already in the original ticket! If you want to verify my email address, just send me a verification link.
1. They usually reply, within about an hour, sometimes longer, usually saying, "We have now adjusted the setting for your server to a more reasonable setting" and not replying to any other question like, "Why is this always happening to us?", "Why did this trigger during the night when hardly any emails are sent".

There was only one time I managed to actually have a conversation with the person on the support ticket. I suspect they were fired for being too helpful. They initially said they couldn't unblock it for me, which, of course, I disagreed with since we have 14 servers, this was the only one to be blocked so that doesn't make sense if we are sending that much bad email. I explained that we send a few hundred thousand emails per week and don't usually have any problems but then randomnly every few months, the servers start getting blocked.

He effectively admitted that the heuristics their system use would treat it suspiciously if a server that was hardly sending anything suddenly starts sending lots of emails. I explained that this is the nature of pretty much every single mass-mailing organisation. We might send a few hundred some days, very few overnight but maybe the next day, we are sending 200 per minute over a number of hours. Using that as a tripwire is complete nonsense. He eventually unblocked the server but it still happens.

## The Microsoft best-practices guide
When you get replies from Microsoft support after they unblock your server, they always rather patronisingly refer you to their best-practices guide. Not a link to the Postmaster site though, an insecure (http) link to a PDF rather grandly called the "Outlook.com Enhanced Deliverability white paper". Now considering how quickly tech changes, how recent do you think this paper is? 5 years old? Nope. It was written 18 years ago in 2007 and they are *still* referring to it in their current support communications.

Just another area where Microsoft seems to have started despising the culture of just "doing what you do well". If I worked in the email team, I would be angry about having a whitepaper that hasn't been updated in 18 years. I would update it myself, it would probably only take a few hours and might actually help people!?

This completely non-modern paper refers to Sender ID Framework and guess what? It is deprecated. So yes, you are being sent to a document telling you to implement something that is no longer maintained and hasn't been for over 10 years. Nice one Microsoft.

## Using the Certified Senders Alliance or "Return Path"
So Microsoft specifically link you to the "Return Path Certification Program" and guess what? It isn't called that any more because it was bought out by validity.com and they are effectively a commerical organisation who MS are suggesting you pay for a potential boost in reputation with MS. The pricing is not visible on the website so you can only "Schedule a Demo". What is it about companies that can't advertise their prices? Are you embarassed by them or are they "flexible" based on how much you think you can charge someone?

The Certified Senders Alliance is better only in as much as it is a not-for-profit. However, it is still quite expensive (although for the volume of work they have, probably not excessively so - how many people actually pay a subscription). Another annoying thing is that they charge based on company turnover, "irespective of how much of your business relates to email sending", which is a bit annoying. We have considered this but there are two practical problems. Firstly, it still provides no assurance that we won't get blocked by Microsoft or anyone else, ultimately, we have no action we can take to force them to leave us alone. Secondly, the requirements are quite strict and we have about 15 things we need to change, most of which are impractical or can't be done until we are sure we are ready to sign-up with them and pay the money. It requires things in the footer that are not entirely clear but ultimately would require changes to how our customers compose their emails, which is not a 5 minute job.

## So what?
Well we use MailJet for all company emails because at least those seem to be delivered OK. One benefit they have is a very large subnet of public IPs (a /16 I think), which means they can protect themselves from being considered "transient ISP IP addresses", which ours are sometimes treated like because the email provider cannot tell whether ours are "static" or just some random thing from Vultr/Redstation that we have spun up for 10 minutes to SPAM people.

The other thing is that I suspect the top people in the large providers are probably all known to each other. I expect MailJet could call Microsoft if they were having problems and discuss a solution whereas small businesses like ours do not have that network.

Which is all pretty sad. I don't mind email providers having rules or limits or even heuristics but it should be mandatory that:
1. Every provider who might block your emails has a feedback registration program
1. They all have to provide accurate information e.g. "You have been blocked for 4 hours"
1. They need to have a point of contact who will tell you why you were blocked so you can discuss whether that was reasonable
1. They need to accept that they cannot stop SPAM and would be better to put stuff in a SPAM folder to be deleted after 30 days BUT, if someone clicks "Not spam", they have much more useful feedback to improve their heuristics and work out why they got a false positive.

Until then, it feels like we can only say to our customers that deliverability to all hotmail, live and outlook email addresses is patchy and there is nothing we can do about it!