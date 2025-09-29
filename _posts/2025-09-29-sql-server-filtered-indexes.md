---
layout: post
title: SQL Server Filtered Indexes
date: '2025-09-29T13:47:01.000+01:00'
author: Luke Briner
---

## SQL Server Filtered Indexes
Filtered Indexes are a really important tool for the Developer/DBA so let's take a step back and understand how and why they solve the problem they do.

## Why do we need Database Indexes?
I did a whole [YouTube video](https://www.youtube.com/watch?v=Pw173bq6mWA) on this for SQL Server a staggering 5 years ago (it seems more recent).

Anyway, I won't go over all of that except to say that a Clustered Index in SQL Server (and other databases) defines the order that the data is stored on-disk. The column(s) that define the Clustered Index are then the "primary key" of the table.

What this means that if you are searching based on the primary key column e.g. `SELECT something FROM tablename WHERE primarykeycolumn = 234`, something very common in CRUD applications, it will be very fast because SQL Server can use a binary-type search algorithm to find the correct row in storage. 

How? Imagine you have a phone directory, something that those of us of a certain age will remember. It is phone numbers ordered by either a person's surname (Personal phone book) or by Business name (Business phone book). If you have ever used one of these or a dictionary or something else, since you know the order of entries, when you are searching by whatever you have ordered by e.g. a surname, you don't start at page 1 and flick through the whole thing, if you were looking for, say, "Jones" you might open about 25% of the way through and depending on where you landed, flick a few pages either way and after no more than maybe 5 attempts, you will be on the correct page and find what you need! Great. This is a SEEK.

What if you were looking for the *name* of the person whose phone number is 01234 567890? It would be very difficult because of the ordering. You would literally start at the beginning and go through every single entry until you find the result (or not if it is not guaranteed to be in the book). This is a SCAN. 

You can imagine that a SCAN is *much* slower than a SEEK so we need to avoid them and that is why we need indexes.

## How does the index work?
Imagine that as well as the phone book ordered by surname, you had another one that was ordered by phone number? Well ignoring the fact that it would only cover a few telephone areas, it would allow you to find a name from a phone number using a SEEK so you get what you need - a fast lookup. At what cost though?

Imagine the two versions of the phone book, you will need twice as much paper to print it and twice as much shelf space to store it, which is exactly the same problem you get with a database index (or not quite: more on that later!). If I want a second index on my table ordered by something else, to allow faster lookup by something other than the primary key, it will cost you double the disk space by default.

This might be a cost worth paying but what if your table is 300GB in size? Do you really want to add another 300GB for an index?

## Options to avoid the size
Why not just set the clustered index to order by something else? Well, this fixes lookups by whatever you want to order it by but it still won't allow you to quickly search for things by more than 1 lookup key. You should also have a clustered index where new items are appended and not inserted. If you ordered by e.g. Surname and you added a new row "Jones", it would need to find an empty row in the correct page of the table and if it couldn't find space, it would need to split a page, copy half of the data to the new page to create space and then add the new row. That is mostly why people use an auto-increment integer or sometimes a UUID that is ordered by time so new ones are always higher up the table than older ones.

The "correct" way to avoid the size issues of indexes is to reduce the columns that are indexed and/or returned from the index. Imagine that instead of all the data for a person, the "lookup by phone number" index just contained a reference number and that reference number could be used to find the data in the original phone book? The new index could potentially be much smaller.

In fact, most database tables are more than 3 or 4 columns, so if your table search function looks up something by "lookup column" and only needs to return a handful of other columns, you can index on "lookup column" and then *include* the columns you need to return. This could be much smaller than the clustered index, maybe only 10-20% on a table with lots of columns. 30GB extra is much more desirable than 300GB extra.

Indexed columns are those that appear in the WHERE clause of your SQL so if you had `SELECT country FROM users WHERE name = 'luke' AND age = 35` then both `name` and `age` would need to be indexed columns but `country` could be included in the index instead.

## Why Filtered Indexes then?
In the above example, you could possibly get quite a small index, but if you need to return most or all of the columns in the query then that is where it can fall down. If you index is fully covering, it will be used but will be really large. If it is NOT covering, SQL Server will work out whether to use the index to get the primary key and then a "Key Lookup" to get the missing columns from the clustered index or if it thinks there will be too many key lookups, it will simply ignore the index and scan the clustered index!

Secondly, if you search by a number of different columns, then you might need indexes for all of them so your 30GB index becomes 10 x 30GB indexes and you are back to eating disk space!

This is where a [filtered index](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes?view=sql-server-ver17) *could* be the solution. The filtered index is like a normal index but in addition to setting indexed and included columns, you can define a filter statement like `Active = 1` or `Age > 20 AND Disabled = 0` and if your query contains part of the where clause that matches this (in any order) then the query planner will prefer your filtered index. It will still be subject to normal optimisation like if it doesn't return all required columns, it might cause key lookups or be ignored but now the space savings could be significant if your have a table where you are querying only a small number of rows using static comprison functions like `Age > 20` rather than something dynamic like `name = 'luke'` which is not what a filtered index is for.

I added one to a table of background jobs to process, most of what are already set to `Done = 1` so my filtered index is simply based on `Done = 0`, which would normally only return maybe max 10 rows. On my 3MB table, it only took 16kb of space, which is probably the smallest it could be anyway and that was an index that was including most of the table columns so that it did not require a key lookup to be performed.

SO for scenarios where it fits, it is an amazing tool to speed up queries without eating up space. The MS page lists these as suitable scenarios:

* When the values in a column are mostly NULL and the query selects only from the non-NULL values. You can create a filtered index for the non-NULL data rows.
* When rows in a table are marked as processed by a recurring workflow or queue process. Over time, most rows in the table will be marked as processed. A filtered index on rows that aren't yet processed would benefit the recurring query that looks for rows that aren't yet processed.
* When a table has heterogeneous data rows (lots of different values). You can create a filtered index for one or more categories of data. This can improve the performance of queries on these data rows by narrowing the focus of a query to a specific area of the table. Again, the resulting index will be smaller and cost less to maintain than a full-table nonclustered index.