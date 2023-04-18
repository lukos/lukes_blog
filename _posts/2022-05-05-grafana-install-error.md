---
layout: post
title: Grafana installation error
date: '2022-05-05T12:32:00.000+01:00'
author: Luke Briner
tags: 
  - Grafana
  - Docker
  - Kubernetes
  - NFS
---

> GF_PATHS_DATA='/var/lib/grafana' is not writable

tldr; You need to set your volume permissions correctly

## "Hello world" examples should just work
Grafana is awesome. It has an enormous rich eco-system of dashboard widgets to watch just about anything in your software stack. CPU over time, RAM, error counts, network usage etc. all very easy. They also have the license model I really like: Use the community version for free but if you want a few more features or support (i.e. you are getting serious about it), then you can pay for an enterprise licence.

However, installing it in Kubernetes is not sadly due to a lack of good documentation. I often lament that documentation is neglected in most companies, it is not produced the brightest and the best so it rarely has that good balance of detail vs simplicity.

Anyway...

I want to install Grafana into Kubernetes and unsurprisingly, Grafana has both an OS and an Enterprise official Docker image - so far so good! Since I am new, I don't want to do anything weird and probably won't need to customise the image so instead of creating my own Dockerfile, I will simply create a deployment in Octopus to use the official image directly from Docker with some config to make it work properly (DB connection etc.). They have an example but it is not really commented so I assumed it would just work. It didn't.

I haven't done much with uids and gids in my containers other than creating new non-root users with whatever ids they are assigned and using them. The example shows fsgroup as 472 and a supplemental group of 0. I didn't know exactly what these meant but the first one is an additional group id that all container processes inherit, it does not have to be a real group id. the second is a group added to the first process to run - this can be useful if the first process does a bootstrap and might need to be part of different groups.

0 is root's UID and group id so you can see why this might be specified for the supplementary group. 472 is the id of grafana in the docker container so it doesn't make sense to also add that as a groupid as per the sample. Instead, you can use it to make the grafana user a member of a group which has permissions to modify the persistent volume where grafana is storing its data.

So without knowing how many processes are actually running in the grafana container, the sample would create:

* Process 1: id 472, group 0, 472
* All processes: id 472, group 0, 472

But let us now turn to the way that *I* have setup my NFS share:

Folder permissions: nobody:plugdev 770

Nobody is a special user that mostly comes into play when we "squash" remote root users into a lower-privilege account. Plugdev is a group that allows you to specify which users are allowed to do stuff with these shares. Of course, you could open the folder to the world but that isn't really secure at all!

My plugdev group contained a single user "ubuntu" with id 1000

When you mount NFS shares, the *names* of the users are not important, only their ids. Sadly, this can mean that if you allow e.g. ubuntu(1000) to access this folder, then any other machine that has a user with id 1000 (which you can specify if you want), can access that folder. For that reason, it isn't great to use default creds but setup specific accounts with specific ids to match each client to the share.

For example, now that I have a grafana (472) user (in a container but that's not too important), I (think I) have two options to get this thing working.

1. Create a user on the NFS server with id 472 and add it to the plugdev group
1. Find the id of the plugdev group on the NFS server and pass this to the container in the deployment script

In option 1, when the remote access is made, the uid of 472 is matched to the local user who has permissions to access the folder since it is part of the group
In option 2, when remote access is made, the fact that the remote user is part of the plugdev group (46 in my case), it is also permitted to access the folder.

I must admit by the time I worked all this out, I was doing option 1 and 2 so I can't promise that both will work separately!

Anyway, it works now!