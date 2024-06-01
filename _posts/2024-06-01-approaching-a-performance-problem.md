---
layout: post
title: Approaching a performance problem
date: '2024-06-01T16:59:00.000+01:00'
author: Luke Briner
---

## Background
One of the common jobs of Developers is debugging something that is slow and hopefully being able to fix it. This is just such a tale but although the more senior of you might not learn much about this, those who are less experienced might well do!

We have an internal web application at SmartSurvey, which we will call Admin and which we use to manage user accounts including the various resources within their accounts. One of the relatively newer features is a page called Review Surveys, which allows you to see which paid features you would lose if you downgraded your account plan to a cheaper alternative. For example, if you want to use the Net Promoter Score question type, you need to be at least on the Business plan, the Professional and free plans don't have the feature. If you reviewed a Business customer who wanted to downgrade to Professional, for example, it would flag NPS questions on any surveys that you use them on and tell you that this feature is available on the minimum plan of Business.

The problem was that this page is slow, VERY SLOW. In our development environment, we have a large account that has around 870 surveys. Not that many compared to some of our customers but enough to test a page like this that processes a lot of data. If you review surveys on this account in the debugger, it takes 2 minutes not just for the page to load the first 10 items in a listing control but also each time you move to another page of 10. If you try a similar thing in production, it will time out so it is basically useless on any large account.

## Why things might be slow
Things like this are a red rag to a bull when I see them. They are just so wrong that I have to fix them even if no-one has asked me to (although I don't know why they haven't, it is so bad). The first question is really, can I easily find out why it is slow and if I can, can I benchmark anything that I can improve so I can see how much difference I can make?

There aren't lots of things that can make something that slow. Ignoring weird or unusual causes like weird lock timeouts on a database, slowness might be number crunching (graphics, hard maths) but we don't do any of that. It could be processing enormous amounts of data but with network and disk speeds so high, it would have to be enormous amounts to be noticeable in most cases (or maybe if we would doing it many times in parallel). Otherwise the three likely issues are 1) Excessive database query numbers 2) Very slow query performance 3) Poor utilisation of the correct collection types when processing data in-memory.

For 3), you might find it strange that this is an issue but I once sped up part of our system from 10 minutes to about 20 seconds by recognising that calling 1000s of `collection.Where()` in Linq was effectively iterating an entire list millions of times and although that work was not strenuous for the computer, it takes a finite amount of time to move through a list, perform a comparison and then store the result. Instead, I converted the data into a Dictionary of key => list<value> and the dictionary lookups being super-fast made a massive difference.

## Using the right tools
I have worked with some people who can answer a question like this very quickly, which we should be able to, but also with others who have all number of ad-hoc approaches. Some people just decide what they think something is and spend ages working through it, sometimes only to find their assumption was wrong. Others just jump between random things to try and spot something that looks out of place but clearly neither of these is that useful. Start with an easy way of ballparking something before getting into details. We can spot the number of, and performance, of database queries really easily in the SQL Profiler for SQL Server, which is what I started with since I suspected this was the most likely problem.

It goes without saying but being able to run the debugger locally and get a locally reproduceable issue is critical. If it is only slow in production, that can be a problem since then we have to make more guesses, perhaps add logging or something, re-deploy, try again etc. The speed that you can run up a debugger and spot the slow parts is a critical skill for a Developer.

## The benchmarks
Fortunately, the problem was easy to recreate and we had a large enough account in our development database to see the problem from the debugger. 870 surveys on an acocunt and the page took 2m05 to load and made, wait for it, 7,600 database queries. Yes indeedy!

The initial one that actually fetches all of the feature information was quite slow (around 10 seconds) but that was subsequently cached so a relatively small hit to take in the scheme of things. No query was super slow other than that, it was sheer volume. Now you might think that database connections are fast, and they should be, but how fast is fast? 5ms? At 7,600 queries x 5ms, you are still looking at 38 seconds and clearly some of these were taking a little more than 5ms. YOu simply cannot run that high number of queries and get good response. If those queries were unavoidable, you should have an architecture that queues the work in the background and allows you to access the results later.

Caching can be a code smell since caching can solve a performance problem but at a cost of RAM and a risk of stale data. Neither was too significant in this case but it was a bit strange to see only one slow query result being cached and the rest of the slow stuff being executed each time. Even if the whole thing was cached though, the initial hit would still be far too slow and would betray a poor design choice.

I had a very low bar to start with so improving it would hopefully be quite easy. On the way, I get to interpret the mindset of the original author, who no longers works for us (not because of this mind you!).

## My approach
When you have such an obvious cause, it is usually easy to make an initial approach. Look for something that looks repatative or unecessary and either remove it or optimise it in some way. Fortunately, the SQL Profiler made it really easy to see the worst two offenders for repetitive queries.

1) For every single survey and feature combination (1000s of them) a query is made to the database to fetch the most recent response on that survey for display purposes. If you have 40 features used on a survey that is 40 requests for the same information, all in the form of a DB query. Now this query requires a "TOP 1" and an "ORDER BY" so it is not a trivial query.

2) For every single survey and feature combination, the code would make a call to the database to basically ask, "what is the lowest plan that supports this feature". In other words, if you had NPS questions on 100 surveys, 100 identical queries would be made to return the same information.

It turns out that with 870 surveys in this example, roughly 7000 database queries are made just for these 2 culprits. If we can reduce these, we can definitely speed up the page load.

## Problem 1 - Get latest response
There is one simple solution for this, which escaped the original author and that is that we load the most recent response date for all surveys we are processing in one query! This does 2 things: firstly, we avoid repeating a query for the same survey maybe 10 or 20 times but also, even if we weren't repeating the query, we would also reduce the round-trip overhead of, say, 3500 queries to the overhead for 1 query - the actual query would all also take less time due to the lack of repetition.

Another super useful part of using the SQL Profiler is that you can see the exact query that is being made for the original code, even if that query is generated by Entity Framework. In this case, the query was correct, a single column queried after ordering the responsedates newest first and then taking the top 1. We therefore have to ask the question of how we can effectively merge all of these into a single query, which is not always obvious (Chat GPT can be quite good at answer natural questions if we don't know the right words to choose) but fortunately, I have used ROW_NUMBER() before as a way to use a Common Table Expression to create the ordering and then a select statement to only take row number 1. The query was something like this:

```
WITH cte AS (
	select
		surveyid,
		endedsurvey,
		row_number() over(partition by surveyid order by endedsurvey desc) as rn
		from responses
        where surveyid IN @surveyids
)
select
	surveyid,
	endedsurvey
	from cte
where rn = 1
```

`ROW_NUMBER()` is clever. The "partition by" is asking, "what do you want me to split into separate groups?" and the order by is, obviously enough, the order that those groups will be in. `surveyid` is our partition so each row with the same survey id will be numbered separately and from newest ended date to earliest. We then get a number out, 1 for the first row, 2 for the second etc. 

Outside of the CTE, we then select the columns we want but nice and easily, we can limit it only to the first row of each group or partition.

Now we can do this in Linq theoretically but just because you can doesn't mean you should. Linq hides so much that even if you can get the correct result set, you still have to double check it is behaving properly and not, for example, pulling back all columns of a table and therefore possibly missing an index. Also, you have to be careful that the Linq expression can be translated into SQL so it can make best use of SQL Server's strength at optimising queries and filtering rows. If not, you can return 1000s of rows (or more) before they are then filtered in memory to perhaps only 10s of results wasting a lot of bandwidth and database time.

We use a combination of Entity Framework Core and Dapper and for explicit SQL, Dapper is pretty easy to use and can automatically bind the results to an object, like EF can. I passed a list of int as the parameter `@surveyIds` and Dapper turns that into the correct format for `IN`.

Once this data was returned and put into a dictionary, instead of querying the database in every loop, we simply did a Dictionary lookup for the most recent date. Sure, the query relative to the old query is much larger since, in this case, the "IN" clause has 870 entries but we are still talking < 100ms and a one-off hit up-front.

> Tip: Using a distinct on the surveyids list ensured I didn't make the query any larger than necessary since there is a maximum limit on IN although it has never been formally expressed, it is around 2500.

> Tip: When you have worked out your SQL, make sure you run it with the query analyzer to ensure it is not doing any work you don't expect and it is hitting an index rather than performing a full scan.

This first change reduced the number of database calls to around 4000 (a reduction of nearly half) and halved the execution time from 2 minutes to 1 minute. Still far too long but much better than before and leaving me to start considering problem 2.

## Problem 2 - Get plan for feature
The idea of this query is to say, if they are using feature X, what is the lowest plan that supports that feature. Again, the author naively treating it as a simple loop is, again, calling the same thing multiple times for a total of around 3500 queries. Again, it would be trivial to make a single query to get all of the potential feature to plans maps and then to access these as a dictionary during the loop.

I could see this being called in the Profiler and made a little dig to find out where I *thought* it was being called from and a top tip is to always prove your assumption. Even in this simple case there were multiple overloads of the method and I could have changed one that wasn't being called! Using the EF extension method `TagWith("text")` you can add a comment to the top of the SQL that EF produces which is an easy way to see in the profiler that what you think is being called is correct.

I say trivial but this is, again, where you have to understand what is already happening and try and understand why. The code *looks* like it was intended to return all plans that the feature is supported on although the query contains a `GROUP BY` and a `First()` so it appears that they are only getting the lowest plan and not all plans. Similar code is used in another application so I suspect that the same types are being used even though in this case, it only wants one plan result, not all of them.

I had to do some refactoring here because these were called from a lower loop and therefore I needed to inject the master list into the method in order to avoid that method making it's own queries. Again, I didn't want to make a breaking change so I simply created a new method that was similar but different.

In a similar way to problem 1, this was a CTE using ROW_NUMBER() but it was slightly more complicated!

I already mentioned that the existing type for the results was not flat but had a property that had a list of "plans" instead of a single plan. The result from the query only contained a single plan so how could I project this into my object? Well, I learned that Dapper supports multi-mapping and has an overload that takes two input types and an output type and allows you to use a lambda to set it. This is the code:

```
var results = contextAccessor.Query<FeatureResult, Plan, FeatureResult>("The query CTE similar to before",
    (result,plan) => {
        result.PlanList = result.PlanList ?? new List<Plan>();
        result.PlanList.Add(plan);
        return result;
    },
    splitOn: "PlanTitle")
    .ToDictionary(result => result.PropertyName, result => result)
```

If we were returning multiple rows with the same FeatureResult values then the lambda would be invoked for each row allowing you to populate the list of plans with each iteration! How does it know which columns are `FeatureResult`s and which are `Plan`s? Using the `SplitOn` parameter, we tell it the first column of the next/2nd type (you can have more than 2). In our case there was only one row per FeatureResult so this was only called once and we could have changed the code slightly (PlanList would always be null) but this is fairly belt-and-braces and allows for returning more plans if we needed to.

Replacing the loop query with the up-front data load reduced the number of calls from 4000 down to 450 and the page load down to 12 seconds. Tonnes better than we started with but hopefully I could make it even better. 

## Problem 3 - Getting translations
This problem was revealed as much smaller than the others but completely exposed once the other two problems were fixed by the simple use of eager-loading!

In the application, when you try and open a survey that has premium features after downgrading, you get a message telling you that you cannot without upgrading. This feature shows the actual question text if the feature is quetion-related, in order for the customer to find which question it is really easily. In a quirk of our system, if you use Net Promoter questions, we don't store the question text in the normal way but simply store an index which relates to "Company", "Product" etc. and which allows it to build the text consistently and with an optional translation for that question in another language.

Due to this complication, the original author wrote something that was functionally correct but performed terribly when used in a loop. It would hit all NPS questions at which point it would query the survey to find its translation id and then it would load the translation for that question and populate the question text. In other words, 200 NPS questions would be 400 database queries.

At this stage, changing it would break it and eagerly loading it seemed a bit too niche to take the hit on so I did something that worked for my use-case: I created another overload of the method that doesn't make any database calls but instead simply puts "NPS" and the question id as the question text.

There is often a problem with reusing an existing solution for a different use-case: something acceptable in a single call that might take a few seconds might be acceptable but calling the same thing in a loop hundreds of times simply multiplies that latency and makes it unusable. Also, this listing doesn't even display the question text but it is still generated as it is needed by the original use-case.

This was a simple change that didn't require a new DB query and removing these calls reduced the number of DB calls down to a paltry 5 from 7600. Once the intial 5 second cache hit was taken, page load and paging was now 2 seconds, down from 2m05!

## Should I do anything else?
At some point, the returns start to diminish and you need to ask whether what you have done is reasonable as it is or whether you should make a bit more effort. At this stage, there were hardly any database calls and what was there was pretty much needed for the code *the way it is currently written*. I don't need to win brownie points or Employee of the Month, I think it is now more than useable but it does beg some questions that the original author should have considered early on. It is shocking that this ever went live in the state it was in. I would rather have not released it at all if the page load was taking even 20 seconds let alone 2 minutes+

This page is a listing control that defaults to 10 items (although that should probably be more) and the code that renders the listing is designed to use `IQueryable` for the simple reason that we can further reduce an un-materialised query to a specific page saving the expense, that is the point of `IQueryable`. However, in this case, the author did not know how to do this (or didn't know there was a problem), he simply materialised the entire thing and then returned it as `IQueryable` so although the paging was there, it was paging an object that was materialised every single time you changed page! This code is more complicated than a simple listing but at minimum, the fully materialised list could have been cached for, say, 5 minutes to at least allow paging to be quick but ideally, they could have refactored the code to actually think about what each page represents and working out how to query those objects using `IQueryable` after which they could run the main queries only to populate that page but this wasn't happening.

Like I mentioned before, caching is a code smell except perhaps in very high traffic sites, so that sort of demonstrated a lack of understanding of what was happening and how to do it properly. Even a stored procedure can be parameterised in some way to reduce the dataset being worked on.

A more fundamental mistake is in the whole reason for the page in the first place. The original feature checking code was designed to warn the customer about features that would be lost on downgrade and show them which features might prevent a survey being reopened on a lower plan. That is all fine. However, when this page was added to Admin, the intent was to provide information for Sales Managers when dealing with downgrade requests. "Can I downgrade from Enterprise to Business to save £3000 per year?" "Yes you can but you would lose the Blah feature which you have used on 50 surveys, is that OK?"

So if this was the case, the implementation which splits out every single premium feature on every survey is too detailed for that use-case. A detailed view might be helpful if pushed but the basic information would be a listing that says Feature X used on 50 surveys, Feature Y on 70 surveys etc. which would be much easier to communicate to a customer but this was not considered and what was produced was therefore not only too slow on large accounts but was not a very useful feature when it was finally released. In fact, I know it wasn't used much when I found a feature that caused it to crash since that feature was not on any plan by default and the code assumed every feature had a plan and fell over!

The page already has an excel export which would perhaps have been more useful for the detailed view and leaving the page to show a summary.

## What did I learn?
1. Even Senior Engineers sometimes produce garbage and don't ask for help to understand why.
1. A couple of hours and some basic tools and you can make something terrible into something good without much risk!