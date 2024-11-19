---
layout: post
title: Beware the slow Linq
date: '2024-11-18T08:47:00.000+00:00'
author: Luke Briner
---

## Debugging slow code (it was Linq!)
At SmartSurvey, one of the features is the ability to "export" survey responses into a number of hard formats, in this case, a spreadsheet in CSV format. We have had performance problems in the past (and still do) with the PDF export since the conversion of the generated HTML is very fast and only the conversion to PDF eats quite a lot of memory and is quite slow (a different story), but generally, these exports take a second or two.

We have a Windows desktop application, currrently, that we run up multiple instances of and which looks in the export queue for the next available export to process.

Since people have a habit of doing things like scheduling an export every Monday morning at 8:30, we do have times where there might be, say, 50-100 exports in the queue, which is fine if they only take a second and if we run 4 instances in parallel. Every now and then, however, it seems to lock up and an export takes far too long, last week, I spotted one that took 20 minutes! If several slow ones hit at the same time, then the entire queue can be blocked for a significant amount of time.

## Luke to the rescue!
This is my area really and I quite enjoy some debugging. We have had two types of performance problems before (one in the exporter and one somewhere else), which were 1) Calling database queries inside of a reasonably sized loop. People think of queries as fast but if they are taking, say, 10ms and you are making 1000 queries, that adds up to 10 seconds just for the queries. 2) Linq performance problems which are actually problems with using the wrong choice of collection when processing large amounts of data. I needed to find out out if this problem was one of these and if so, which one and where in the code?

## Debugging is not a common skill
I have read a number of blog posts that talk about debugging a problem and am very impressed with people tracking down very subtle bugs. If I was better at debugging, I wouldn't be as impressed but I know lots of Devs who would take a long time and use poor debugging tools like logging to file to try and capture some data points (how big is this collection, how many times was X called etc.) and may or may not eventually find the problem.

## Debugging whether it was a "query in loop" problem
Debugging the potential "query in loop" performance problem should be easy. I know how to run the SQL profiler and filter it so if I just run up the exporter locally and use our very useful "test export" feature, which allows us to raise exports which won't be processed by the normal exporter but allows us to pick them off them debugging. The only problem was that this issue was seen in production and we can't run up the profiler against production (according to the CTO) since it slows to a crawl, although I don't know if he used the filtering features on it at the time.

Attempt 1: Add logging to our database calls in the exporter and log them to a file locally while pointing to the live "problem" export. Doing this with Entity Framework was actually surprisngly easy but this didn't yield any loop queries. Doing this to Dapper was harder but only because of a historical reason to have 2 constructors in our repo classes and not having direct access to the Dapper connection to replace it with a decorated version with logging. Hmm.

Attempt 2: Try a similarly large export on the Dev system with any "variables" I could spot in the live survey that might cause a problem (like filtering or the File Upload question type). This ran very quickly so I hadn't identified the actual issue so this wasn't a useful test case.

Attempt 3: Copy the problem survey response data to the dev database so that I *could* enable the SQL profiler and see what was being sent. The problem here is that we don't have a great mechanism to do this (and generally never pull live data into the dev database for privacy reasons). We can create copies of the relevant tables with the relevant data in but he script that is generated automatically by SQL Server to "Generate Scripts" simply uses IDENTITY_INSERT and puts the data directly into the target, which often fails due to conflicting primary keys. Not an easy workaround so this was also a bomb!

Attempt 4: Perform the same process on the Disaster Recovery (DR) site. Fortunately, the DR database gets an automatic copy of the production database each night so it would already have the data. The DR web server also has a copy of the exporter so I ran everything up. This worked and very quickly proved there were no loop queries. This had taken too long but it did remove a fear that I had that somebody had added some code without thinking about what happens when a survey has 65,000 responses!

## Debugging slow Linq
Anyone who has formal training in Computer Science will learn about algorithms and data structures and why every collection is not equal even though their memory usage might be similar and they offer similar ways to achieve the same thing. Unfortunately, in the UK, a lot of Developers have no Computer Science degree or sometimes any formal training and are not familiar with these foundational concepts, the result is code that "works", but is orders of magnitude slower than the correct choice of data structure. This is missed during development with small data sets but occurs more or less frequently in production when it is more likely to hit the largest amounts of data they are being used for.

The exporter loads everything needed for the export from a query into memory and then loops this to write into the output (in the case of any "all responses" export types anyway). Traditionally, these just use .Net Lists, the easy goto for any .Net storage needs. You can add, take away, search, etc. unlike an array, the data doesn't need to exist in contiguous memory so mutations are fairly cheap (although we are not mutating it so who cares right?). "Use Lists for everything!"

Well, my next step in finding if this was a problem was using Jetbrains dotTrace profiler. Profilers are so useful but traditionally, there aren't many around and those that exist are not that user friendly. I understand a lot about memory management, time-in-function and what functionality might be CPU hard or I/O hard but I still struggle to navigate these tools, although dotTrace is probably the friendliest to use, some of its docs are out of date so the tutorials refer to options that are now deprecated.

There are two questions you might be asking with the profiler: "What is using up all this RAM?" or "What is taking so long?" and these have preset options in the startup dialog so that is all fine. You need to get everything ready before starting the trace because you don't want it collecting the time that your app is waiting for user interaction, this just makes the results less clear because it will contain a lot of "non-user code".

I did this and after fumbling around slightly, I found some code that said a lof of time was spent in the constructor of a DTO? A DTO with nothing in the constructor! But after clicking the next line up, I saw this inside the loop that writes each user row to the output:

`attachments.Where(a => a.QuestionId == question.QuestionId && a.ResponseId == responseId).ToList()`

This is a good example of something that reads clearly and "works" correctly but has terrible performance on large datasets (it is poor on smaller datasets but not necessarily worth fiddling with if only used in small datasets). So we have loaded all of the attachment data (file uploads) up-front from DB into a list and then for each response, we are simply finding the attachments for the current response row and current question column, which are then passed to the rendering method.

Let us consider what is actually happening at a low level here, considering this survey has 17 questions, 65,000 responses and nearly 70,000 attachments. This will create a list of 70K items, which will then be accessed 65,000 x 17 times = 1.1M times! When using a `Where()` rather than `FirstOrDefault()` is that we have to always traverse the entire list looking for each value we are filtering (question id and response id) to know whether to return it. There is nothing clever in List that remembers anything from the last time you looked. You could probably use a sortable list to make it a bit quicker but there is a better way (later!).

So we will need to run 1.1M x 70,000 comparisons checking `QuestionId` and `ResponseId` for matches = 77.3 *trillion* checks. Can you see why this might be slow? Even on a massively fast machine and even if the check only takes microseconds, this will not be good.

## How to fix this?
The code has to be fairly generic, in that there might be multiple attachments per question and per response so any type of sorting or assuming `FirstOrDefault()` will have minor benefits at best and will be wrong at worst. The correct way is to ask how we could store this in a way that is faster to retrieve. This is where the `Dictionary` comes in. This powerful collection might seem a bit niche since it maps some kind of key (often a string or number but any object can be used) to a value (often an object but might be a simple type as well). various "Hash" type collections use a similar mechanism but don't have key, the object is its own value.

The power of these Hash-based collections, including Dictionaries, is how they are used to perform the lookup when you access e.g. `myDict[myKey]` which seems like it might just iterate all the items in its list like a List and then return the matching one but if it did, it wouldn't be much use. No, what the Hash collections do is they...wait for it...hash the key (or the object for hashsets) into an integer, which then using mod arithmetic is used to find an index in an array. They use mod so the index is circular. The size of the internal array is important for performance so you should always create a dictionary with a given capacity that is at least roughly the correct size for the data set. If you create one from another dataset, this is done for you.

For example, in our example, we could create a dictionary of `QuestionId => List<Attachment>` or `ResponseId => List<Attachment>` whichever one we use, we then still need to perform a single filter on the result we get back. Note that if we used the first example, since this survey only has one file upload question, the dictionary would have a single entry with a List of 70,000 attachments, so that would be useless for us. Instead, by pre-separating the users into separate lists, we are likely to get much smaller lists (0, 1, maybe 2 items per list) so filtering on the questionid will be super fast.

So our dictionary, when created from the database Linq query, will have an array of roughly 65000 size, each of them with a small list. When you access e.g. the list for user 123456789, an integer doesn't actually need hashing into a number since it already is a number! What will happen is the collection will simply perform `123456789 % 65000 = 21789` and return the list at `internalarray[21789]` which is amazingly fast!

In fact, the export that was taking nearly 20 minutes, now takes about 5 seconds!

## So what?
Debugging is an important skill and understanding collections is essential to writing acceptable code. If the Developer didn't even appreciate what is happening under the covers, they might even assume, "it is what it is: large surveys = slow exports". In fact, one of my colleagues a few years back asked me why his export utility (something else) was taking so long while using a very large list. It was good that he asked for help because he could have struggled on and wasted time. I was able to teach him about the search penalty for using a list with large data sets.

Anyway, make sure you are training your teams and at least pricking their curiosity about things they don't already know.