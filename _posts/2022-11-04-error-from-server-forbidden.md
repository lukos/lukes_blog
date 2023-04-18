---
layout: post
title: kubectl apply - Error from server (Forbidden)
date: '2022-11-04T13:23:00.000+00:00'
author: Luke Briner
tags: 
  - kubectl
  - ConfigMap
  - Cloudflare
---

## kubectl apply -f ./my-config-map
Not doing anything fancy. I have 4 configmaps to upload to K8S and 3 work fine, the other produces the error "Error from server (Forbidden). There is some rough HTML which doesn't say much more than "There was a problem with your request".

No hits on Google (weird) and not really any hints until I looked at the URL I was hitting for the cluster and noticed that it was shared with the admin portal for that cluster and that the domain name is proxied by Cloudflare.

Yes! Cloudflare had blocked this request simply because it contained SQL statements which rightly would have looked suspicious. What was disappointing though was that there was no clue in the message that it was a Cloudflare proxy error that might have given me a clue as to what was happening.

It could easily have taken me many hours to track down the problem (I first saw it about 2 days before I fixed it but had left it to fix later).

Reminds me of the old adage that "Obscurity is not security", which is true but here's another one: "Give the user information so they can deal with the problem". Not quite as pithy but important nonetheless.