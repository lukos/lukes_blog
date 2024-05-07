---
layout: post
title: NPM in Docker is a dumpster fire
date: '2024-04-24T12:43:00.000+01:00'
author: Luke Briner
---

## Background
Trying to run a `gulpfile` via `node/package.json` inside a `Dockerfile` that is building a dotnet 8 application.

Easy right? Nope.

## Ubuntu Jammy 22.04 + aspnet 8
This is the base image for the build/deployment (the sdk image has build tools but not relevant here). Of course, it doesn't include `node` or `npm` out of the box because why would it? It includes the minimal installation to build and run a dotnet core app.

### Option 1 = apt-get install node npm
So this works easily enough and installs these packages and although npm installs an astonishing amount of dependencies, they were only needed for building and I might be able to worry about that later. The problem, as you might expect, is that the package versions are *very* old. Node 12 in fact, which was EOL in April 2022. Not surprising since the LTS version of Ubuntu was released in April 2022 and LTS don't like updating major versions of things which cause breaking changes. Which, of course, is generally OK except node isn't like everything else. The release cycle is very short: 18 months active followed by 18 months of maintenance. Anyway, I can moan about these old versions but I can't really do anything about.

Ran the rest of the `Dockerfile` including yarn install and I get an error, basically, one of the npm packages requires a newer version of node (v14, also EOL a long time ago but not installed by Ubuntu) so this option is no good. Hmm.

### Option 2 = nvm
`nvm` seems great but only because node is so bad at handling multiple versions. It is an open source utility that can download and install multiple versions of node side-by-side and you can then select the one you want to use either now or for all future invocations of node. Seems to be a good candidate for updating my node versions.

Except: Docker!

Run the install script in Docker and that works but then you get "nvm command not found". Why? Because Linux and it's weird way of updating paths etc. `nvm` is not actually a binary but a script pretending to be a binary, which is fine if you are using it interactively. Why? Because it adds some stuff to `~/.bashrc` that will make it visible to your shell. If you look at the docs, you can either logout and login again for `nvm` to appear or you can even `source` your `.bashrc` file to reload it without the logout. OK.

`RUN <nvminstall script> && source /root/.bashrc`

`source: not found`

## Trying to get source to work
As I keep going further alongside, it feels like I am taking a car apart to do something that should be easy like changing the stereo unit and there are various suggestions to try and get around this issue. What is happening is that Docker is using `/bin/sh` as the default shell, which is not bash and doesn't, therefore, support the `source` command.

### Option 1 - Re-link the shell executable
One particularly hacky solution, related to Docker, was simply to remove `/bin/sh` and then create a symlink from `/bin/sh` to `/bin/bash`. Someone else said this was a terrible idea since other scripts would expect `/bin/sh` to be POSIX compliant and since bash is not, this is likely to cause other problems.

Anyway, this didn't work

### Option 2 - Set the SHELL
You can try things like `SHELL ["/bin/bash","-c"]` to set the shell for all of the rest of the script. This has a similar effect to the previous hack and ideally you should only set this for the commands that need it and then change it back to `SHELL ["/bin/sh","-c"]`

Also didn't work. `nvm command not found`

### Option 3 - use the login switch
Then I read that the whole `.bashrc` thing is related to an interactive login which is what you would get with something like `ssh` but not inside Docker. If you do not use the login, there is no `.bashrc` so you can't source it, but you can pass "-l" to the `SHELL` command to make it interactive.

Still not working. It now gets past the NVM section without error but then `npm install` fails again because "you need version 14 and I found version 12". Just like `nvm` hadn't run at all even though I could see the Docker build log downloading and installing a newer version of node.

## Maybe I need to update npm too?
There is a symbiotic relationship between `node` and `npm` even though they are supposedly independent. If you download node, you usually get a specific version of npm included and although this is probably just to derisk it, it does create a lot of confusion. Can I use a really old version of `npm` with a new version of `node`?

So then another rabbit hole. You can use `nvm install-latest-npm` which supposedly finds the latest version of npm supported by the currently selected version of node. Sounds good in theory, except you need npm in order to get the latest npm, which wouldn't be so bad, I installed it from apt remember, but the next annoying error, "cannot determine npm version" or something like that. It was like the whole nvm world was separate than the normal world and they were not talking to each other. Of course, I had installed npm as part of the same build process so maybe it was a similar issue to `source` where the nvm process cannot yet see/read the apt npm version.

Instead, I thought I could use `npm` to update itself instead of using `nvm`. It should be able to find itself? Tried that too: "newer version of node required. Require v16, found v12". npm doesn't seem to have a method to say, "download the latest one for the node version I have", although since the apt node was so old, maybe it was already the latest npm version it could handle and npm wasn't seeing the downloaded node v21 from nvm for whatever reason.

## Eventually I got to the end of the build layer
Initially, I didn't even know if everything was actually working (some simple scss compilation), I just wanted to get the tools running then I could debug, I thought if I could get to the end of build, I would have enough to start working out what was or wasn't working correctly.

I had ended up setting the bash shell right at the beginning of the build layer since it seemed like each change of shell lost visibility of the previous changes so keeping everything inside the bash shell made it look like it was working.

Except not! A really strange thing was happening on the last step, which was unchanged from before, building the `csproj` file using `dotnet` cli. When running this command, it spat out all kinds of random errors related to files that were in the repository but are not referenced in my project (or solution). They are in the repo because they are part of a submodule. How on earth could this be right? I wasted another few hours scratching my head and getting frustrated.

## Going back to the start
It's sad when something is such a mess that you spend a whole day on it but then have to go back to the beginning. If I build the original `Dockerfile` it's fine. So let us try and add things in one-at-a-time and see where it goes wrong. We don't need to run any gulpfiles or whatever, let's just try and get it building.

So that is what I did. The most likely problem was changing the shell to bash so I decided that I had to work out how to get things working without using `source` since that was the only thing that was needing the bash shell.

Using some of the techniques listed later, I found what `nvm` downloaded and added to `.bashrc` and realised that the easiest (but horriblist) way of working around the issue is that every command that needed node needed prefixing with the code that loads the nvm script and so a line would look like this: `RUN . "/root/.nvm/nvm.sh" && nvm install 21.7.3`

It was weird that I needed to add that each time I ran something like `yarn install`.

Anyway, after far too long, something that should have been much easier than it was ended up being a big learning experience. I already hate most things about `node` and `npm` and this was just more ammunition.

## Debugging tips
Apart from the usual tips about using layers and cache correctly and understanding enough about Docker and (in this case) Ubuntu so that you know roughly what is happening, here are a couple of other things.

### Debug the state of your container
If you are seeing that commands are failing etc and you don't know why, get your build to complete to a certain point without running the failing command. You can also use `--target` to target a particular stage in the `Dockerfile` to avoid building everything. Once that image has built, if you try and run it from Docker Desktop, it will immediately exit unless it has a `CMD/ENTRYPOINT` *but* if you get its hash from the image list in Docker Desktop, you can run it like `docker run --rm -it <image hash> bash` and get a container that you go into and then using command like `ls` or even running the failing commands to see what output you get. Sometimes, errors are simply because you got a working directory or copy command wrong, others are a bit more involved.

If you can get the commands to work in your debug container, you can then update your Dockerfile and potentially build to the next error.

### Build your own base images
This is a bit more work but if a lot of your apps are based on e.g. Ubuntu + node 21 then you might as well create a base image and start with that. It will save having to solve the same problems in multiple places as versions change and applications update.

### Consider your own apt cache server
I was having such random performance problems with `apt-get` (>30 minutes to update and upgrade) that I used `debmirror` to setup my own apt cache for Ubuntu 22.04 which only takes around 380GB of space but you can manage your own performance then and not have those days where the archives are a slow as mud.