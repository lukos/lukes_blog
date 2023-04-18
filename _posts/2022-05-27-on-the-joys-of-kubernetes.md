---
layout: post
title: On the joys of Kubernetes
date: '2022-05-27T17:37:00.000+01:00'
author: Luke Briner
tags: 
  - Kubernetes
  - Ingress
---

## What are you really getting from microservices?
Lots of discussions about microservices quickly go to "why use microservices if you are not Google? Just use a monolith". Now this is partially true but I would argue this is an in-between, one that we have at SmartSurvey but only for certain things. Email for example, is sent in a batch. We could use a single desktop application/service to send all of this but that presents various challenges, not least that this would only run under one account unless it runs as "Network Service" then it would need a separate UI to connect to it. If that machines goes down or needs to be rebooted, we need to run up another instance, close the first one and start the second. Not great but that was what we used to do.

What I wanted to do instead was to have something make sure that my batch processing is always running, that I can easily restart machines without playing the dangerous "did I break production" game. I also want a system that will scale in the future as we go from sending 30K emails per day to perhaps 100K or 1M, it would be nice to have co-operating systems that can share the load between them.

Container orchestrators offer the automatic monitoring part but we also get some benefits by using a managed Kubernetes service (we use AKS but the same is true for most of them):

1. A lot of the complexity is hidden for you
1. Upgrading the kubernetes version is point-and-click
1. Adding more nodes is point-and-click
1. Updating the nodes to newer/bigger machines is point-and-click
1. The price is only for the node VMs

## With great power comes great responsibility
I am definitely more of the "move fast, break things" culture so I am also very good at upgrading Kubernetes versions (it's point and click remember!) but what I don't usually do is to check what breaking changes are present and what I might need to do. This is partly difficult because we use Octopus Deploy for deploying our containers so I don't really have to play with yaml too much.

It also means that when things do go wrong, it can be hard to debug although that is more related to Kubernetes rather than managed Kubernetes. Most debugging consists of kubectl and grep!

## 503 Service not available
I deployed an existing service to AKS which previously we ran on our own Rancher cluster. I pointed the DNS to it and got the dreaded 503 error from nginx. At least it shows you the error so you know you are hitting the load balancer/ingress. In Kubernetes however, this error can mean a number of things 1) Your pods are simply not connected properly e.g. service misconfiguration, mistyped paths etc. 2) Your pod is erroring and falling over causing nginx to send you the error instead of the pod 3) The pod is not available e.g. liveness probes are not working or it hasn't started 4) If you have deployed the parts separately, you might not have matched the service selector to the pod selector.

I tried all of these, comparing them to another working service and they all looked correct/the same.

Fortunately, when I went onto Github looking at another bug, I saw something helpful, "Attach the ingress controller logs". Oh yeah, the ingress is actually a set of controllers that look for ingress objects and updates nginx with the config so it works. I find they are in the default namespace (in my case) and see that 1 is running but the other 2 have restarted loads of times and are constantly crashing! So this was the cause of the 503. Since they weren't working, they were not reading in my newly deployed service and therefore it was unavailable. The old ones hadn't changed and had been read before so the existing services were all working fine.

## Debugging the controllers
Fortunately, because the ingress is run by controllers as pods, you can simply `kubectl logs nginx-controller-xyz -n default` and see the problem: 

```reflector.go:138] k8s.io/client-go@v0.21.3/tools/cache/reflector.go:167: Failed to watch *v1beta1.Ingress: failed to list *v1beta1.Ingress: the server could not find the requested resource```

Followed by sigterm!

What I knew I had done yesterday was upgrade Kubernetes from 1.21 to 1.22 so I dashed over to the upgrade page and found some information. Surely enough, the `v1beta1.Ingress` had gone from deprecated to unsupported in 1.22. Following the very verbose instructions about "Preparing to upgrade to 1.22", I first had to check whether my current ingresses were compatible with the updated spec. (I didn't use any of the other advanced stuff that had changed fortunately so just the one job). I had about 10 ingresses and fortunately, they already contained `apiVersion: networking.k8s.io/v1` as the version and `ingressClassName: nginx`. These come from Octopus so it must have either read the version of K8S to choose the correct format or maybe they just updated it to use the new stuff. One thing that WASN'T up to date was that the rules were still using `ImplementationSpecific` as the type, which apparently is legacy. These all needed to change and since mine were all just routing everyting by path, I could change all of these to `Prefix` but I then had to redeploy everything and try and make sure that they all still worked before the ingress could be updated. They did but I'm not sure if they simply didn't reload because of the crashing controllers or whether prefix was correct (it was!).

## Upgrading nginx

Now that the ingress definitions were all up-to-date, it was a case of upgrading nginx. Unfortunately, when I first installed AKS, I followed their instructions and used Helm to install nginx because that's what they said to do. It has been a bit hairy because being part of the load balancer, any mistakes can take the microservices down! What is also VERY annoying is that there is an `ingress-nginx` helm chart from Kubernetes and also an `nginx-ingress` from nginx. You have to be careful because they are not the same and it is easy to find yourself on the wrong website.

It said to add an `ingressClass` if upgrading to Kubernetes 1.22 but if you installed nginx with Helm, don't do that otherwise the upgrade will fail!

I found using `helm list -n default` that I had the kubernetes one installed so I went to the page and checked out how to upgrade. It suggested `helm upgrade --reuse-values nginx-ingress ingress-nginx/ingress-nginx -n default` (being careful that you use the correct name as returned from `helm list`, the correct namespace for your system and the correct name for your local helm repo!). That didn't work!

## Helm upgrade error
`Error: UPGRADE FAILED: template: ingress-nginx/templates/controller-service.yaml:1:53: executing "ingress-nginx/templates/controller-service.yaml" at <.Values.controller.service.external.enabled>: nil pointer evaluating interface {}.enabled` Nice right? A really clear error. This is related to Helm but simply-put, it means that you are trying to reuse values that are not valid any more. If you didn't add any customisation (I didn't) then you need to remove `--reuse-values` from the upgrade command. If you did, then you will need to do your homework and work out what to pass to the upgrade command to set the custom stuff on the new version.

Once that was all done, Helm upgraded properly, the controller pods all started behaving and more importantly, I could still connect to the microservices, including my new one!

So much joy for one day!