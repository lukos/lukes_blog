---
layout: post
title: Understanding documentation is the real skill
date: '2022-05-06T20:20:00.000+01:00'
author: Luke Briner
tags: 
  - Software engineering
  - Job skills
---

## What is the most important skill for a Software Engineer?
You might have some ideas about this, maybe it is:

* Deep understanding of algorithms
* Good at turning the problem domain into the software domain
* Super pragmatic
* Business minded

Whereas all of these are important skills to have, if my past few days has been anything to learn from, the most important skill is the ability to use documentation!

You know when you interview people and they say that technical questions are bad because, "I would just Google it"? that's nonsense. Not because we don't Google things but because searching and applying documentation is hard. Very hard. Most engineers cannot do this as well as they claim as if Google = instant answers.

Why am I thinking about this? I have just setup Prometheus to track metrics across our production systems and I had a number of issues around some basic things:

1. NFS permissions on a Persistent Volume causing errors in the Grafana "getting started" deployment.
1. Similar issues with the Prometheus data directory
1. Trying to working out how to add a file to the Prometheus config volume before the pod started so that it wouldn't crash
1. Trying to work out using Azure storage as Persistent Volumes - either manually or automatically
1. Trying to understand how to get Prometheus metrics out of Rancher
1. Trying to understand how to get Prometheus metrics out of Azue Kubernetes Service

So why was all this so hard? A combination of things that I suspect most of you experience on a daily basis:

1. Assumptions made by documentation e.g. you have already set the correct NFS permissions, which if you don't know about them, you fall on your face
1. Documentation with missing information e.g. Prometheus doesn't tell you where to map a volume to persist data across deployments - surely something you would *always* do in Kubernetes
1. Azure has sooo much documentation about things. They have umpteen different types of storage accounts with a large number of options (NFS/Samba, Files/Disk, Premium/Standard, Ephermeral/Retained, Blob/share/container) that working out which one to use is hard
1. Really long detailed documentation. If I have to follow 10 steps to do something, include a script on the page that I can run. Don't make me copy and paste a tonne of snippets
1. Kubernetes is hard as it is so when layered on top of something else, makes for super complexity. e.g. Persistent Volumes + AKS or Service Accounts + Helm
1. Lots of out-of-date articles that never get cleaned up from the internet and which search engines don't *seem* to downgrade over time since they are less likely to be accurate
1. Helpful people offering solutions that are not the correct solution, which might or might not work and which can lead you down the garden path, "For me, I restarted Visual Studio and the problem went away"
1. Articles that look helpful until you realise they are written by companies selling add-ons
1. Things like Azure Monitor that are trying to sell me something I can get for free with Prometheus and which mask the fact that I don't need their commercial offering
1. People who tell you steps but don't explain what each part is doing and why it might or might not be needed
1. Sarcastic responses to genuine forum questions
1. Being closed down by Github maintainers because they are too precious about their baby project
1. Old tutorials (Grafana!) which are based on old versions of the application and haven't been updated as part of the massive development work to update the application itself
1. Documentation that is trying to maintain multiple versions of an application but which are not SEO friendly and which you struggle to navigate between the versions (Rancher!)

In the end, I have got where I intended but it has taken around 3 days of web searching to do things that are supposed to just work. I have been a developer for 20 odd years and it was hard for me, it will be hard for less experienced people!

## What is the solution?
Documentation needs to be a first-class citizen for any software supplier or organisation not the necessary evil that lends to half-done un-maintained documentation. It needs to include regular testing and feedback from customers needs to be treated with reverence, not disdain. The aim should be zero-problem documentation. I fed back some information once to MS about an article that was confusing (I don't want to sound big-headed but if a 20 year veteran is struggling, there probably needs to be some more information) and nothing got done for 6 months! That's right, the documentation team were so understaffed/non-agile that they did nothing. After 6 months, they closed the issue since they were changing the documents!

Customer feedback, especially unsolicited, is valuable - it means someone else is reviewing what you did and telling you it falls short - listen to it, provide prompt feedback, if you must dismiss it, do so politely, not with that degrading tone of, "Well we don't think this is a problem".

And last top-tip for interviewers, if you want a genuine skill, give someone an interview question that requires them to research what they are doing on the internet and then check their solution. This will tell you whether they really can, "just look it up on Google" or not!