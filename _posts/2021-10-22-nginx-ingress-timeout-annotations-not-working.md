---
layout: post
title: nginx ingress timeout annotations not working
date: '2021-10-22T19:46:00.000+01:00'
author: Luke Briner
tags: 
  - Kubernetes
  - nginx
  - annotations
---

Kubernetes is very complicated. The basics are not too bad and you can make something work simply without knowing too much but it doesn't take long until
you move off the beaten path and the documentation is...well...variable. Like all modern framework/tooling, the documents generally cover several versions
including new features you might not support yet, old features that are already deprecated and a million combinations of questions and issues on various
github projects and forums, "Why is X not working on GCE?", "Why does Helm support X and not Y?". The answers are usually all in there somewhere but they
could be part of a Tutorial, part of the API docs or even just a footnote, "You must use network driver X on Windows".

## The Problem
I have a service that handles a file upload, parses the spreadsheet and then uploads the contents to a database. If the spreadsheet is 450K rows+ like one I
was working with today, the service, understandably, might take a minute or two to complete the operation. Nginx default timeouts are 60 seconds, a not too
unreasonable default.

The service runs in a pod and we have nginx ingress in front of it, like many Kubernetes clusters.

Now the documented way to solve this is to add annotations to your ingress yaml to override the defaults. All very sensible and supposedly easy. Except mine
didn't work. After 1 minute, the gateway would time out. I checked by logging into one of the nginx controller pods and cat'd /etc/nginx/nginx.conf and sure
enough, the various proxy timeouts were set at 60s and not the 180s I was trying to set them to. What was interesting was that another annotation to set the
max request size *was* correctly set so my (incorrect) assumption was that there was something special about the timeout annotations.

## The debugging
I did read the documents and I found two common gotchas:

* The value for the timeouts has to be a *string* but also an integer (all annotations are strings). Therefore, unlike some other settings, you can't use e.g.
"60s" or "3m". You also can't use an integer like 60 or 120, it would have to be only a quoted integer e.g. "120" - mine was correct (or so I thought)
* If one of your annotations is invalid, the ingress can ignore all of them. This is important because we might not notice if we set an incorrect annotation until
after adding a second or a third, whcih don't work. I double-checked mine and all was well.

I was actually deploying from Octopus deploy and one of the handy features is a verbose log that shows the yaml that gets built by Octopus during a deployment so
you can see exactly what is being sent to Kubectl. This is where the penny dropped.

Octopus already quotes the values you enter in the various annotation fields, since they are always strings, you never have to quote them. Because I had, the value
of the timeouts wasn't "180" but "\"180\"" which was obviously invalid and was therefore ignored by nginx.

A simple mistake but a reminder that fault finding is possible by breaking down what you are doing into the smallest testable steps. I was actually really close to
raising an issue or dumping the question on Stack Overflow but I decided that since I couldn't find many instances of this question being asked, I decided it was
more likely to be my setup and not the system.