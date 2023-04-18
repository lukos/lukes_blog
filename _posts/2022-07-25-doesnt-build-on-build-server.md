---
layout: post
title: It doesn't build on the build server
date: '2022-07-25T11:38:00.000+01:00'
author: Luke Briner
tags: 
  - Msbuild
  - .Net
  - CI
---

## How many times does it work locally but not on the build server?

Firstly, you need to decide whether it has never worked on the build server or maybe it has stopped working recently/it builds on some build servers but not others!

WARNING: Not all errors surface as errors, only as warnings. This might cause either a build to go green that hasn't actually done what you think it has or otherwise it might have failed later on in the build than it would have done if the warning was an error. When checking builds, you should look at all of the warnings, at least once, to ensure they are expected and aren't any nasties!

## It has never built

If you have a build log, it is usually reasonably obvious what has failed in the build but some clues are more subtle! Start by checking any errors logged, which might show a direct
or indirect problem.

### Example: 'BackgroundJob' is not declared. It may be inaccessible due to its protection level.

This could be a simple compiler error, although it builds locally right? It might also imply, as it did in this case, that a nuget package didn't restore correctly.

So what basic checks can we make to fault-find?

* Clean your local build, check your output directories are clean (not everything might have been cleaned) and then exit/restart Visual Studio and perform a rebuild of the solution and check that it definitely builds locally
* Make sure that you have not forgotten to commit any files or you have forgotten to push. This might not be obvious if you have something in your gitignore that shouldn't be there
* If there are files that are environment-specific, have you ensure the build server has its versions present and correct?
* Do you have any environment variables that are relevant on your local machine that have not been defined on the build server? Have you made sure they stored against the machine and not against the current user?
* Nuget will require an outgoing network connection from the build server so make sure that step works - the error output should be clear if not.
* Your build server is likely to run as a system or special user so if you install things, they need to be installed for all users or under the build server user.

## MsBuild vs Visual Studio

It is likely that you are using msbuild/.net cli when building on your build server and Visual Studio locally. There are a significant number of differences with msbuild requiring more switches than VS which will do certain things automatically for you.

Things to consider:

* If you are using custom nuget sources, you will need to inject them into msbuild e.g. `/p:RestoreAdditionalProjectSources=https://api.nuget.org/v3/index.json` for msbuild
* If you are using dotnet core, you can combine a nuget restore with a rebuild on the server by passing "Restore Rebuild" as targets. However, in older style projects, this won't work and you will need to run a `nuget restore` step (with your custom nuget sources if any).
* There are versions of msbuild which nominally relate to the versions of visual studio but do NOT generally match the internal versions of tools so be careful. For example, MSBuild Tools 2019 might actually be version 15 on the disk. It should, however, support everything VS2019 does, as long as you installed the correct workloads in the installer.
* Different versions of MS build tools etc. supported various technologies that stopped being supported so if you have any hard-coded paths in your builds (which hopefully you don't), these might have worked on an older build tools but will not with newer ones (or the directory will need changing to match where it *now* lives!)
* You can always copy the msbuild command that is created by your build server (it should be in the build log) and run it locally on your machine. If it doesn't work, it is more likely you will spot the problem. If it does run locally, it is more likely that you haven't installed something on your build machine.

## It works on some servers but not others

The most likely issue here is that you have some tools or SDK installed on one machine and not another, you can check that in "Installed Programs" by comparing but not all problems are solved in this way!

* A very annoying and subtle bug is if there is a nuget problem but on one server, it has cached certain packages from previous builds and so has no problems, the newer server, however, doesn't have the packages. Nuget warns instead of errors when it can't find nuget packages, which imho is very wrong and should be changed since you are likely left with build errors, which don't make sense.
* Out of band applications that you might run like IISExpress might not be installed but the build server won't know that unless it is a built-in step which can check the agent for pre-requisites.
* We have had problems since we run one agent on the build server. If we have a step that e.g. copies the output to c:\output, which exists only on the server, then the build works on the server's agent but not on any other agent. This should be quite obvious from any errors.

## Conclusion

Since automated builds are based on an enormous number of variables, there are certain things you could do to avoid/reduce the problems:

* Docker moves the build to the developer machine so it *should* build on the build server if it builds locally. As containers get more complex, however, it is possible to still get a problem centrally so you should be defensive in coding, for example, erroring if a variable has not been set.
* Automating your build agent deployments/updates with something like Puppet/Chef/Ansible can ensure consistency (I don't know enough about all of them and which is a better choice, probably the one you already use!) If you are super slick then even updates can be made this way and automated to avoid the manual steps which we can get wrong!
* Even a simple checklist that is kept up-to-date for creating new build servers but also when you are updating servers, you can keep a ticklist of what needs doing and on what machines it has been done on.