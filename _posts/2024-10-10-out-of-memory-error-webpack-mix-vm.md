---
layout: post
title: Out of memory with Laravel Mix/Webpack on build agent
date: '2024-10-10T09:56:00.000+01:00'
author: Luke Briner
---

## It works on my local machine
A Docker build running yarn, which runs Laravel Mix, which uses webpack under the covers. My change was simply to split the Mix pipeline into two sections that could be invoked separately using an environment variable just to make the Docker caching work correctly and not run the 2 minute Mix build for any changes to C# app code.

Runs locally, exactly as expected. Push the branch and merge and the CI build errors.

`error Command failed with exit code 1.` is all I get. What a piece of crap! I know any exit code that isn't zero is an error, but what error? There was no obvious reason why my change would break anything so let's just run it again. Nope.

## Where do you start when you "know" you didn't break it?
* A colleague started by trying to add more debugging to webpack and running it again. Nope.
* Another build agent? Nope.
* Run the Docker build directly in the build agent. Same problem.
* Run the `yarn` part from the Dockerfile directly on the build agent. Same problem
* Try increasing the `max_old_space_size` option for Node (Urgh). No dice

## The clue
Sometimes I saw an error 137, which apparently means Out of Memory, so that's something to follow. My local machine has tonnes of RAM, of course, but the VMs have up to 12GB which should be more than enough

I then look at the Hypervisor running the VM while yarn is running and the RAM allocation, which is dynamic, never goes above the starting RAM of 4GB. Weird, maybe it isn't a memory error then, otherwise it would request more.

Or would it....?

## Testing the hypothesis
I shutdown the build agent, set it to start with 8GB instead of 4GB and - voila - it worked.

Curses! It seems that Docker builds that need more memory do not request it from Hyper-V when running in dynamic RAM mode. The only problem with that is that by upping all of the build agents by default, we now don't have enough RAM on the hypervisor and our provider want loads of cash to increase it so I will need to sort out my cloud-based Team City on AWS where at least you can run up beefy servers on-demand and leave them shutdown overnight to save money.