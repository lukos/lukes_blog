---
layout: post
title: Nullable types slow down dotnet core
date: '2024-03-18T13:40:00.000+00:00'
author: Luke Briner
---

[Nullable reference types](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references) are a feature added in C# 10. It seems a bit weird because reference types are always nullable so what is this feature about?

The problem with what Microsoft call "oblivious nullable state" or the previous scenario where all reference types could be null or might not be null, is that the compiler cannot help you spot the places that you might be de-referencing a null reference. That means that the programmer is responsible for considering all cases through code where something that should be non-null *might* be null and will throw if left in-place. On the other hand, the programmer might be adding an excess number of null checks for things that will never be null due to the code, which just adds more noise.

Marking a reference type as explicitly nullable like `string? myvariable;` tells the compiler that I know this might be null so please help me to find places where I am de-referencing it before checking for null, which by default is a warning but can be elevated to an error if needed. On the other hand, if I do NOT mark it as nullable, it means that I am telling the compiler that this should never be null. If I do that, the compiler will see any places where the variable might not be set before it is used, which would cause a `NullReferenceException` to be thrown.

It's all pretty helpful but it has an annoying (but understandable) side-effect in ASP for dotnet core. If your viewmodel has a property that is not marked as nullable, the framework helpfully makes the property as implicitly required! That's right, even if you haven't added the `RequiredAttribute`, the property will be required and will fail modelstate validation if not set on POST. This is annoying for me since porting an MVC 5 app to dotnet core, it has broken a lot of validation and it is hard to find where this will happen without testing every single action. I could switch it off but if I do that, when will I ever switch it on again? Whenever I did, the problem would immediately surface again.

However, although that is annoying, it has a knock-on effect that you might spot on more complicated pages like our feature setting page that has about 300 features listed and which you can enable or disable on a per-customer basis. I was trying to fix an unrelated bug on the page and was testing the validation of the message you type when you make a change by using the required attribute. I added the attribute, ran the app up, left the box blank and clicked "Submit". It didn't seem to work....no wait...it is working. What? Let's try again, leave the page come back, press submit and after about 3 seconds, the validation message comes up and this is Javascript!

You might see where I am going but I did scratch my head for a bit until I looked at the html markup to see what I could see. All of the 300 dropdown lists that contain the feature status were marked as required. This wasn't affecting the posting of the form because they are all set to their initial value regardless of whether they are changed i.e. there is no "not set" value. However, the application of the `data-val-required` attribute was causing all 300 of them to be validated on submit and for reasons I do not know about, this takes about 3 seconds before the message for the input appears.

There is always something that makes me uncomfortable about non-obvious problems like these. Implicitly making something required breaks the principle of making errors obvious since it won't always have an affect and if it does, and you don't have a validator rendered, the page simply fails to submit and you have to work out how to debug what is going on.

That said, I don't really know what I would do instead, perhaps just allow us to disable "nullable types = required" in model binding and let us live with the consequences of where we might accidentally forget to set something. Not sure