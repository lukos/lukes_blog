---
layout: post
title: Why you must test database queries
date: '2025-06-04T14:03:00.000+01:00'
author: Luke Briner
---

## How many Developers equate "it works" with "it is correct"?
Scenario: A Developer is asked to add a new page into a web application that loads a list of widgets and perhaps has some search functionality. They add the page, it looks OK, the code looks OK and the search appears to work. Many Developers stop there.

The page goes into production and it ~~runs like a dog~~ (strange metaphor because dogs run quite fast) runs like a dog with no legs. Some then complains that the search doesn't work in all scenarios. Ouch, what happened?

The Developer saw that the page "worked" and that the search "worked" and decided the job was done. A good reviewer would have asked, "Did you test the DB query in the profiler?". Even if the Developer tries to be defensive, "Oh, it's pretty much the same as the other code that was there before", my answer is usually, "I don't care about the code that was there before, I care about the new code. Go and test it in the profiler". Why so arsy? Because there several ways in which you can be caught out if you don't!

## Things that go wrong in database queries
1. The test database is much smaller so the queries seem to run at an acceptable speed when the Developer tested it but they didn't know that the query was missing an index and performing a table scan. In test test DB with 1000 rows and not much contention, it is fast enough. In production with 1000s of connections and 100s of millions of rows in the same table....yeah. Really, really slow and possibly also causing deadlocks for other connections as a result.
2. Your query is "pretty much the same" as the old one but it is not the same. You added another column in the select statement which is not in the index but you didn't notice. By profiling, you spot the problem and update the index or change the way you query the data.
3. Your query is projected in a strange way. Your where clause on a nullable column, for example, that is basically `Deleted != true` gets projected as `Deleted <> cast(1, as Bit)` which means it misses the filtered index for `Deleted = false` and again causes a table scan. If you notice that, you change your code and find that using `Deleted == false` projects correctly.
4. Your search term has not been passed correctly so it only finds fields beginning with something rather than any string within a field. You didn't notice because your testing was lazy and didn't try a few scenarios.
5. There were too many columns returned in the projected `Select` statement meaning that you will pull far more data than you needed, even if it is quick, it will hurt the network bandwidth. You realise that a certain operation in your Linq-SQL query didn't work as you anticipated and you therefore have to change it to something else or use raw SQL instead.
6. The profiler shows you that the query is performing a key lookup instead of being fulfilled by a single query. This will generally be much faster than a table scan but still quite slow for large result sets. This has happened because the extra column you added is not in the non-clustered index so it decides a key lookup is best. Unfortunately, this can be even worse in production because it will base the query plan on the first result set, which might be small = key lookup but then users whose list is much larger get a dis-proportionately slower experience.
7. Probably others that I can't think of right now

## How do you check it?
A really critical skill of the Developer is knowing their debugging tools and being able to run them up quickly. For example, I wonder how many .Net Developers could quickly run up SQL Server Profiler and see what they need to see and how many either have never used it or would spend 30 minutes getting it working. Not acceptable. Testing a query should take a minute or so.

Run the profiler up, capture the query being sent from the app, run it in SQL Server Management Studio with the Show Execution Plan button pressed. Easy. Understanding the query plan, again, should be easy for any Developer worth their salt. Knowing *why* something isn't working is sometimes harder. There are cases where the query planner takes a wrong decision but 90% of the time, you should easily see why an index was missed during the query.