---
layout: post
title: Setting up Amazon SES with SNS notifications
date: '2026-01-21T15:22:01.000+00:00'
author: Luke Briner
---

## Amazon SES
When you are sending lots of emails, managing reputation is an ongoing and almost unwinnable game. Despite Microsoft having reputation tools that show our servers as delivery very low-levels of SPAM, they "heuristics" will still mark servers as "yellow" and they are often blocked for random amounts of time. To be honest, life is too short and MS are completely unwilling to engage on what they are doing and why other than offering a "whitepaper" from 2007! A multi-billion dollar company cannot get one of their team to spend a couple of hours updating it.

So we decide to use Amazon SES for a number of reasons:
* Pricing is very reasonable
* They have some really great features like "tenants" to separate customer reputations from each other
* Lots of functionality in the form of APIs
* Automatic notifications of issues like bounces or "sending paused" which we can hook into our own systems

It is the last of these that took up too much of my time today!

## Amazon SNS
Simple notification service. Like most AWS products it is not simple but it is reasonably priced, especially for webhooks, which let's be honest are not expensive. But it does take some fiddling. It defaults the payload to text/plain, which makes sense for plain notifications where the message is just text but not so much for SES notifications which are actually JSON blobs in the text property.

The problem was, I couldn't get SES to send notifications at all. I could manually publish a message, which was sent fine. I looked through all the docs I could find, even copied an example from another region that worked but mine still didn't.

The first thing I noticed was that I had the default access policy which might have worked but I suspect not. Even though my colleague said he didn't touch his, mine was set to the default whereas his wasn't. This is documented (https://docs.aws.amazon.com/ses/latest/dg/configure-sns-notifications.html) but I wonder if you create the topic from SES, it might do that for you whereas if you do what I did and create the topic up-front, you have to do this bit (which is fine).

The second part I had got wrong (and was different from my colleague hence why his was working) is that it is NOT enough to link the configuration set to the tenant and just specify the tenant when you said. This sends the email OK because the configuration set is linked but it does NOT generate the notifications unless you also specify the configuration set. For example in the API call:
```c#
var rawRequest = new SendEmailRequest
{
    FromEmailAddress = from.ToString(true),
    Destination = new Destination
    {
        ToAddresses = new List<string> { request.MailTo.TrimEmailAddress() },
    },
    Content = new EmailContent
    {
        Simple = new Amazon.SimpleEmailV2.Model.Message
        {
            Subject = new Content { Data = request.Subject }
        }
    },
    TenantName = $"customer-{actualParentId}",
    ConfigurationSetName = "Default"
};
```

The problem with this, apart from the fact that there are no errors and the emails are sent! is that you need to know, in-advance, which configuration set name to use for which tenant. This is OK if you only have 1 but if you want to specify different delivery policy, different reputation or validation options etc. you will need to create more than 1 and then keep a record of these at your sending end. Again, not the end of the world but fails the principle of least-surprise!