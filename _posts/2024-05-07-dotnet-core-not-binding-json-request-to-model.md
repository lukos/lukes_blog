---
layout: post
title: Dotnet core not binding a json request to an action's model
date: '2024-05-07T20:08:00.000+01:00'
author: Luke Briner
---

## Background
You know when you do something that you are sure should just work out-of-the-box but it doesn't and you scratch your head?

I was trying to post a json request from a Javascript file to a dotnet core MVC action and expected it to bind to the model automatically. It didn't and I didn't know why!

## Problem 1 - \[FromBody\]
Prior to, I think, dotnet 6, json model binding would work out of the box with nothing special. Set the content type, make sure the model matches and boom! However, this was changed "for security reasons" so that now you need to explicitly state that the model is `[FromBody]` so that it will check the content type and invoke an appropriate binder (Json is configured to work by default).

It is also not possible to pass parameters by body *and* also by query, the model binder won't like it and the query parameters will not be set.

## Problem 2 - The whole request needs to be valid Json
This is where I tripped up. I had a model like this:
```
public class MyModel
{
    public int Id { get; set; }
    public List<Response> Responses { get; set; }
}

public class Response
{
    public int Qid { get; set; }
    public int Oid { get; set; }
}
```
As far as I could see, the json was being created in Javascript (I used the `debugger;` keyword to step into the code in the browser) but it wasn't working. I wondered if it was case-sensitive so I changed the properties to be lowercase but no dice. The model was always `null`.

Logging to the rescue! Dotnet core logging is probably the one biggest improvement over netfx, which had almost no logging. I set the level to debug and ran the code again: "Model binding failed due to invalid Json" or something like that.

I had mistakenly used a string representation of the Responses and passed it to JSON.stringify() which effectively double-encoded it and meant it didn't work. Even though it was only the `Responses` property that was broken, this meant the entirety of the json request was not valid and none of the properties would bind correctly!

You also get no errors, just a null model.

So there you go! Make sure your Json is valid, make sure you use `[FromBody]` and make sure you use dotnet core logging!