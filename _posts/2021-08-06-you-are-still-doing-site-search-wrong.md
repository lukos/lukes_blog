---
layout: post
title: You are still doing site search wrong
date: '2021-08-06T11:26:00.000+01:00'
author: Luke Briner
tags: 
  - Web
  - Search
---

Imagine that you ran a shop on the High Street. A customer comes in to buy something. However, despite their effort, they cannot find what they are looking for. They come up
and ask you whether you have a Bicycle Wheel but you don't know exactly so you offer them a wheel for a car, a bicycle and a puncture repair kit and possibly 100s of other
items. They say thanks but no thanks and leave. The thing was, you did sell bicycle wheels but you missed out on a sale, another $10 profit to help you pay the bills and the
only reason was that the sales assistant was unable to do the job. You are probably thinking that you would definitely sack them, or at least realise they need a lot better
training.

If you have already read the title, you probably know what I am going to say because the search on most web sites that I visit, including the big players, is generally really poor.
It is like a bad sales assistant. Some of you might use tracking to see that people come to your site but don't buy and in most cases, you will not know exactly why this is but
I can guarantee that a big cause is that people can't find what they are looking for.

## Examples of poor search experience
Here are some very poor customer experiences:

1. You search for something quite specific and it returns 1000s of results. This is not just demoralising but can make the customer assume that if a specific search does not return
specific results, why bother continuing?
1. You search for something that you are pretty sure that the shop sells and it comes back with no results. This is more confusing when you eventually navigate to the product and
see that it does exist. If you can't trust the search, maybe there are better value items for sale that you can't find and maybe you should go somewhere else
1. You search for something that you expect to return a lot of results, perhaps "wheel", because you are not sure exactly what you want and you not only get the expected 1000s of
results but no way of filtering these down to the specifics like 200mm diameter or cost less than $20
1. Forcing the customer to page through results by clicking a button, that likely takes a few seconds to load the next page. How many times do you think they will click it before
realising how slow it all is and leaving?
1. Returning lots of results due to exact, partial, full text results but seemingly not ordering them so that a puncture repair kit is returned when searching for "wheel" because the
description says, "you will need to remove the tyre from the wheel..." even though exact matches are in the results further down
1. Search results take longer than 1 second for the page to be usable (ideally they should be less than 500ms)

I would think that along with poor service (like emailing a site and not getting a response), poor search is probably the number one reason that people leave your site. Either they
can't find what they want or the process you expect them to follow to find it is asking too much.

## Why is your search bad?
Now you might have bad searching because it came with the framework you used or it is what your developer cobbled together but you cannot sit by and accept a poor search mechanism
on your web site.

You might also protest that search results have to look through a lot of database content, and with paging, this can be a bit tricky. Your site might be a bit slow anyway.

Maybe you have simply never put yourselves in the shoes of your customers and if you haven't, you need to!

## How do we resolve it
Firstly, you need to understand the type of customer who will visit your store and what they might search for and how. You then get the basics correct befre looking for edge cases that 
you can also solve. Let us consider a model railway store, since that is where I have been looking recently. Let us ask how would an ideal search work from a model railway customer's point
of view?

Well, what might I be searching for?

* A manufacturers part number. Maybe I am looking for a locomotive R3867 - I would expect this simply to find it directly and possibly even go into the product detail page if there was only
one result. If it was mentioned in the description of another item, I am unlikely to want that search result unless a) the site does not find a direct match for the code or b) the site
gives me some signposting like, "we also found other products that mention R3867"
* A more generic search term like "southern steam locomotive" - The word southern is important here, otherwise I wouldn't have typed it, it would need to find an all words match for this
search term. It can always say, "we found some products that match some of the words" but this should not mask the fact that the main search failed since I am looking for something quite
specific.
* Perhaps I want to look at some general things like "OO gauge track" and this I would expect to return quite a few entries. Since in many cases HO and OO track is the same size, the site
needs to understand this and again portray it to the user in a clear way, at least if there are no direct results, we might (like Google) say, "did you mean HO?".

### Dealing with typos
Now there might be some challenges here and you should take a leaf out of Google's book since they already show you some tricks to make search work. You should have a system that keeps track
of common typos, "staem" instead of "steam", you can then say, "showing results for steam". This system will need to be maintained since each day someone might mistype something else. If a human
can guess what they meant to type, you can simply add a new entry like "looc" => "loco"

### Synonyms
There are often synonyms that can be useful to include in a search. If I search for "steam locomotive", I might need to search for "steam loco" as well since not all descriptions might include
the full word. Again, this needs maintaining since you might need to run some sanity checks against your stock details to either correct loco => locomotive in a description or to add the synonym
to the search results.

### Always prefer what the user searched for
There are three scenarios to consider. If they search for something and it returns no direct or indirect results, it is easy "Sorry, we didn't find a match for...". If you can suggest anything like
common searches, then great but otherwise that is OK. However, you MUST log failed searches so you can examine them and understand what people might be searching for. What if you realise that lots
of people are searching for "Harry Potter railway" and you don't stock it? Maybe you should! Maybe a failed search is a site or description problem. Maybe you typed in the stock description
incorrectly.

### Images are really useful
Keep search result images small but a basic image is much more helpful than "image coming soon" or "generic image". Of course, a generic image is better than nothing but make sure it is obvious
to the user. They might be looking for a DCC decoder that looks like the one they have in their other loco but don't know what it was called. The right image helps them to confirm their choice,
especially before paying $100 for a new one!

### Speed is everything
Imagine if every time you asked a sales assitant something, they waited about 3 seconds and then answered. You wouldn't stay long. For some reason, we expect people to wait while searching or
paging through results. If your framework or site is slow, get your developers on it, if they cannot make it fast, go somewhere else. Honestly, the cost of reading the bulk of your data into
memory and serving results directly from the cache is not even worth bothering with. Even if going to a database, your queries should be 200ms tops and potentially much faster.

### Use lazy loading instead of paging
Paging is slow because it usually involves an entire new request to a web application and reloads the entire page. Lazy loading via an API means only the results are being appended to and also,
you are not expecting people to click buttons that are often impossibly small, on a mobile device. More results? Just keep scrolling and the results will keep coming in.

### Consider full text directly in the search box
There was a car parts website I saw where although the UI was very dated, the search box was awesome, you could type something like "1995 Jaguar gearbox back plate" and the autocomplete would
keep up with you, allow you to click directly onto the part you were looking for instead of loading search results, hoping to get the correct results and then clicking through. It takes a bit more work
but again, this is web dev 101 today and if a developer cannot do this, you should look for another developer.

### Filters are second-best
Some sites give you back a large list of results and then let you filter them. For example, you might click on Screws at screwfix.com and are immediately taken to a search results page with 1000
entries. You open the filters and start selecting length = 50mm, material = galv steel but you are making someone do a lot of work that they might not be able to, despite the fact that also each
change is probably another page refresh which can take seconds (screwfix, your site is terrible for this on mobile). What about if I know steel and 50mm but not all of the other options? Do I care
about head-type, manufacturer, single or double-thread? Filters, like paging are also a pain on mobile devices and don't always work so, again, ask what the customer will look for. If they search for 
"50mm screw", simply take them to a generic 50mm screw and show them underneath, "other screws are available with slightly different properties". Yes, this might be product-specific but yes, this
might also mean that your customers won't either go elsewhere or even clog up your stores asking questions they could have answered automatically on a good site!

## Always be monitoring!
The trick to any well performing web site is to monitor it. How many people visit, where do they drop-off? What searches are they making? What searches produce no results? What words are often
mistyped? Have you tried it on mobile or tablet? If you haven't, you might be surprised to find it is garbage on a mobile. Some sites are tiny because they scale to fit a mobile, others don't
even work! Buttons that you can't click, text that goes off the side of the page so you can't read it.

A website needs to be an operational expenditure for a business in the same way that you accept paying 100s per month on renting a shop and paying the bills, yet some people are running old web
sites from 20 years ago that simply don't perform well enough. You might get away with it if you are already busy enough but any business that you are losing is probably going to someone else and
one day, if you are not careful, that competitor will be able to outsell you in every respect and you will simply become another casualty of capitalism!

## Where to get help
If you are/were running your own web site or have something that simply isn't performing but you don't have the expertise to work out how to fix it, find a few local development companies and find
out what your options are. Some off-the-shelf shopping sites based on Wordpress, Shopify etc. are not necessarily massively expensive and effectively you can pay a few hundred per month (or less!) to
let someone else worry about how to run a web site properly!