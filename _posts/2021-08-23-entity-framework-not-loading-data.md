---
layout: post
title: Entity framework not loading data
date: '2021-08-23T13:46:00.000+01:00'
author: Luke Briner
tags: 
  - EF Core
  - EntityFramework
---

I would like to think that after many years of .Net and Entity Framework development, I would quickly debug any problems with database access and data but this one took a little
while to get my head around. I was running an integration test that was updating the database for a specific object and then reading that data back in the test to check the update
had taken place. The database *was* updated but my check was failing and reporting that the data wasn't updated.

TLDR; Entity Framework object caching was causing the problem!

## Entity Framework is designed for Unit of Work pattern
One thing that people often ask or get confused by is, "how long should my DbContext exist for"? The dirty answer is that it should live only as long as a single unit of work so in
Asp.Net, this means scoped to the request and if used in a long-running program, which might not have scopes, it should be transient - 1 instance per use.

The DbContext is very lightweight and only does important stuff when you actually use it so don't be afraid to create one transiently. It does not setup an entire connection etc.
on construction.

There is, however, another important reason for making it scoped or transient. Caching.

When you query or load an object using Entity Framework, that object is read into EF's cache. It will NOT be reloaded unless you force it to or if EF knows that it has changed. What
does this mean? If you read an object initially and then another DbContext or something else like Dapper was to change the underlying object, your initial DbContext will not know and
will assume you still have the right one and why reload it in that case?

Why cache at all? Simply to allow developers to be carefree about what they want to load and not worry that things will be loaded 100 times from the database. If you are within a single
"unit of work", you should not expect external changes for the duration of your UOW.

This comes back to the idea of making DbContext very short-lived. The longer you let it live, the more chance that you will miss external changes or *even worse*, you might overwrite external
changes with your own update (although EF does track columns that it thinks have changed so you will not always do this).

## So why was my test not working?
This what my test effectively did:

1. Create a new model
1. Use my scoped DbContext to save it to the Db as a new record
1. Call an endpoint on an API that updates the database record with some new data
1. Read the object back in my test and check for the update

So this should have worked right? If my DbContext was scoped, which it should be, all changes would have gone through the same channel and therefore all would be visible to everyone. Except
that wasn't what I was doing.

I am using the WebApplicationFactory to run my integration tests and therefore I do not have a request scope in common with my API with which to share a DbContext. I create my own test scope,
resolve a context from there and use it. When the API is called, the middleware will create a *separate* scope for the web request and the scoped DbContext is therefore a different instance.
The update would be visible to the API context but not to the one in my test scope.

What can I do?

Solution 1) There might be some way to hook into the same scope as the API web request or perhaps pass in the test scope to the API so that it uses the same scope
Solution 2) Use something other than DbContext to read the data back so that the create and read are not happening in the same context.
Solution 3) Reload the data from the DbContext by calling `myDbContext.Entry(model).Reload()` followed by loading it the second time