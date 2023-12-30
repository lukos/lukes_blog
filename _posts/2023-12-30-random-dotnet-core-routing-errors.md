---
layout: post
title: Random dotnet core routing errors
date: '2023-12-30T12:42:00.000+00:00'
author: Luke Briner
---

## The scenario
I am working on a migration of an older MVC5 (.Net Framework) app to a newer dotnet core 8 app. The Visual Studio Migration wizard does an OK job of creating the dotnet 8 app alongside the MVC5 app and then you can select individual classes/controllers/views etc. and "Upgrade" them which involves it being copied to the new app and some basic transforms made to it, like replacing `System.Web` namespaces with `Microsoft.AspNetCore` or whatever.

There are a number of things it cannot reasonably do automatically so you then need to pay down the debt of poor technical decisions since if you used nasty things like `WebBag` or various other fiddles with `HttpContext` then you are likely to be scratching your head or doing endless identical changes to make it compile.

## The routing problem
The way the upgrade works is that it adds `Yarp`, which is a software reverse proxy which can forward any un-mapped routes to the old app via a localhost URL. All you have to do is make sure that logging into the new app also logs into the old app and then it should all be grand.

It was basically working OK and I have made hundreds of changes but then I noticed something weird recently, that was not related to any recent changes but since I was not always running the app up it wasn't clear at what point it broke.

The problem was that if I clicked onto various menu items or indeed, the dashboard link (located at /), a random action was invoked and displayed and the `Information` level logging in the terminal window indeed showed something like, `Request: https://localhost:7100/` followed by `Executing endpoint: /Reports/ReportTool/Index`. What? It's not even a close match.

## Debugging route problems
What I *should* have realised/tried very early on was to increase the logging level to Debug (using Serilog in my case) since the problem would have been much clearer. I did not however, and attempted to debug into the framework code, which isn't too hard but the code is very complicated and some of it is optimised away so I only learned what I found out later but with no more insight: There were two routes that matched the request and as it had "found" the one inside the `Reports` area first, that was the one that was returned.

Now the debug loggin was much more specific (although there is quite a lot of noise too!) but it said essentially, `Matches the route: /home/index using the pattern {controller=Home}/{action=Index}/{id?}` and also `Matches the route Reports/ReportTool/Index using pattern {id?}`

Ah OK, I can see why it would match a route that has a single optional part called `id` but how on earth did this happen? None of my routes in `Startup.cs` that use `MapControllerRoute` looked like that.

> There are a couple of nuget packages designed to help you debug routing but one is based on the .Net framework Web API and the others don't really give you very much other than confirming the route that has been chosen. The Debug logging tells you pretty much everything you need to know.

## Two types of routing
Then a small bell rang in my head because this has bitten me before, there are two types of routing in dotnet core: Conventional routing and Attribute-based routing. Attributes can have unexpected effects on Conventional Routing and that must be what happened here.

The problem with the MS documentation is that it either tends to be over-simplified without the details we want, or it is so complicated, you have to read it several times to understand it. Microsoft Learn have helpfully separated out "Routing", "Controller Routing" and "Attribute Routing" into separate documents but it feels like it could do with a refresh and make the basic points clearer before diving into too many examples and details.

My app did have conventional routing setup correctly for both root-level Controllers and also area-level Controllers so it should have just worked but I had made a mistake by using the `HttpGetAttribute`. This is an important attribute (along with its siblings) to help the router decide which of two actions with the same name are selected, usually with a Get/Post pair for MVC views. However, there is a subtle trick which is not obvious, when I used the overload that allows a *route* to be included in it. So a very abbreviated copy of my class was like this:
```
[Area("Reports")]
public class ReportToolController: Controller
{
    [HttpGet("{id?}")]
    public IActionResult Index()
    {
        return View();
    }
}
```

The `AreaAttribute` on the class is fine, and is necessary to automatically wire up the area-level controllers but the `HttpGetAttribute` of specifically the one that takes the route parameter is the problem because it implicitly behaves the same as this:
```
[HttpGet]
[Route("{id?}")]
```
The RouteAttribute is the problem because it has context dependent behaviour. If the *class* also had a `RouteAttribute` then it would be combined with the route on the action *unless* the action route starts with a ~ or a / in which case it is treated as an absolute path, something that might be helpful for weird endpoints like `/login` which would otherwise require a customised controller route to be mapped.

However, in this case, there is no `RouteAttribute` on the controller class so even though it does not start with ~ or /, it is still treated as an absolute route instead of potentially either throwing an exception "Route is not specified as absolute" or perhaps just a plain 404. Instead, it is creating this additional route with a pattern of {id?} which, of course, is very greedy and was grabbing most other pages and being very confusing.

## Conclusion
Understand routing and enable debug logging if you get weird routing errors!