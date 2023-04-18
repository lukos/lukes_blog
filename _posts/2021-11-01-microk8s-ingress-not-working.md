---
layout: post
title: Microk8s ingress not working
date: '2021-11-01T11:52:00.000+01:00'
author: Luke Briner
tags: 
  - Kubernetes
  - Microk8s
  - Containers
---

Following on from the hours I wasted trying to resolve the issue with Azure key vault, I now had a service that worked fine in our Rancher cluster in
Production but on DR, it didn't work. DR runs microk8s, a supposedly simple to setup Kubernetes cluster.

The error was one of those weird SSL ones like "record too long" or something. I was definitely hitting the correct endpoint because at one point, it
seemed to be working OK.

The first problem was that the ingress module wasn't enabled on Microk8s. It was frustrating because at one point DNS and ingress had both been enabled
but at some point (maybe the same point, both were disabled), I wasted some time on that but running `microk8s status` on one of the nodes eventually
proved that the nginx ingress controller was working. Fortunately, since it was working on another environment, I wasn't playing around with all of the
settings in Octopus Deploy to make it work.

I also found the answer on Stack Overflow.

For reasons best left to the maintainers of microk8s, they have overridden the ingress class name from the default "nginx" to "public". That fails the
principle of least surprise in my opinion, especially since it is not documented on the main document (sadly, again, documentation severely lags the free
feedback you are getting from the customers!). If you ask for feedback, use it and use it quickly, otherwise people will stop giving you feedback.

So we had used the ingress class "nginx" on all of our other environments to make sure the ingress is only picked up by the nginx ingress. This works fine
except on Microk8s. I don't know how to disable the class on the daemonset, it would perhaps need me to create my own so instead I had to use Octopus 
parameters to ensure that a classname of "public" is used on the DR cluster and "nginx" everywhere else.

Annoying but at least fixed.