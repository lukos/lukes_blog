---
layout: post
title: Visual Studio warning triangle
date: '2022-09-16T15:58:00.000+01:00'
author: Luke Briner
tags: 
  - visual studio
  - dependencies
---

## So you have a yellow warning triangle?

The icon is instinctive right? It tells you that there is a problem with a dependency but it does not tell you what that problem is! How annoying that it can detect a problem but not spend a little time pointing you in the right direction!

What does the warning mean? It means that for some reason, the reference is broken/incompatible or a duplicate. 

Why does this happen? Some possibilities:
1. A project is moved
1. A nuget package gets changed/deleted from nuget
1. You have changed the framework version of a project
1. You have accidentally a duplicate reference using Resharper or VS Intellisense
1. Old build folders hanging around
1. VS not detecting things changing in the file system e.g changing git branch
1. Dirty horrible nasty VS metadata files getting corrupted/out of date for whatever reason

How do we find out what is actually wrong? Just by working through the reasons below:

1. A common issue is simply that the dependency is broken - it is pointing to the wrong place. Depending on the type of project you are using, it might be a project or package reference in a csproj file or it might be a hintpath to your nuget packages folder. You can see where it is looking by right-clicking the assembly and choosing "properties". Open the location it is pointing to in Explorer and see if it is correct. Sometimes you can correct by removing and re-adding but sometimes you need to manually edit a csproj file to correct it.
1. I have also seen this where a nuget update updates the version of the package in packages.config but didn't update the hint path in the csproj. When that happened, it was all fine until it tried to build on a machine that didn't happen to have the old package already downloaded!
1. If you have tried everything else, a less common cause is if you have somehow added an assembly reference as well as a project (or less often a nuget) reference. The assembly reference will be preferred, if you F12 a type, it will open the object browser instead of your code file. Again, you can probably just remove the assembly reference.
1. Make sure that the versions you are using are compatible. For example, netstandard up to 2.0 can be referenced from netfx 4.x but 2.1 cannot even though the numbers aren't very different.
1. Check your nuget references after updating a project to a newer version since it might have a reference to a version of the package used by the old version and not the new version. MS packages, for example, often follow the 3.1, 5.x and 6.x patterns of the main frameworks.