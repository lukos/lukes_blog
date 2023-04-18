---
layout: post
title: Writing a postfix milter in Rust
date: '2022-03-01T17:11:00.000+00:00'
author: Luke Briner
tags: 
  - Postfix
  - Milter
---

## Using Postfix
If any of you have ever installed and configured postfix, you know it is *really* hard. This is due to a number of factors. Although it is venerable and works very well, it is also a relic from 1998 so its featureset is pretty stale, requiring a number of other modules like dovecot, cyrus, opendkim etc. to be plugged into it to do things like TLS connections, login management etc.

At least I think it does because the documentation is very old-fashioned and hard to digest. In the style of many a wizened Unix hack, it is pages and pages of text. That was great in 1998 because who needs graphics right but nowadays, we can quite easily use some basic styling to make things readable. To draw our attention to the important and the "advanced". But no, this is not how it works.

You can search for help online and you mostly find old/incomplete articles and since the configuration is so extensive, you can easily go wrong and not really know what you are doing. I even bought the Postfix book but it only really covers postfix and hasn't been updated in 18 years even though I know postfix has. Things like TLS are ubiquitous now, they weren't when the book was written.

I managed to get all of that setup and working but it is something like 50 different steps to install and configure everything. I am writing a bash script to automate it, something I did before but lost it :-(

## What is a milter and why do I need one?
The problem with postfix being so old and the documentation so hard is that once you need to step outside of the very basics of sending email, like bulk-sending, you quickly get problems. One of these is VERP. If you are bulk sending, and you get a bounced message, you want to know. How could you do this? You could try and parse the returned message but it is not always returned. Instead, ideally, you need unique "Return-Path" addresses that you can extract instead. These can all be routed to a common inbox and you can have a tool that reads the message sent to e.g. user123456789@example.com, which means a previous email sent to whoever user123456789 is was returned.

So? Postfix doesn't really do this. It has *some* VERP capability but it is pretty much fixed and involves something like converting an email sent to e.g. luke@gmail.com to a return path of luke=gmail.com@example.com. This is really basic and doesn't allow us to send multiple messages to the same person with unique tracking links.

I need a milter - a Mail Filter, which is the only way to achieve some kind of deviation from the functionality of Postfix without changing the source and rebuilding (no thanks). With this milter I can do something quite easy like set the "Return-Path" to whatever is passed in for the e.g. "X-Return-Path" header since by default mail servers are supposed to set the return path themselves, even if you pass it in as a header.

## How do you write a milter?
That is a very good question. For a while, I could find no answers. I found one very old example that didn't describe anything about how to build/install it. SendMail who own the milter API are now owned by some corporate so their documentation seems completely unavailable, old broken links, no definitive API description. All very terrible really.

Until I discovered a [Rust Milter](https://docs.rs/milter/latest/milter/) crate. I had already found something in Python but I don't really know Python or Rust so had skipped past both. Then I realised I didn't have much choice and took a look at the Rust one.

## Why Rust?
I took a short foundations course on Pluralsight but realised that in many ways, Rust is not massively dissimilar to C#, although it does have a few quirks related to safe memory management which are worth learning. I only wanted the basics so I could understand the example code like the arrow that defines the function return type and the Some/None matches. I have already done lots of C and C++ back in the day so the pointing referencing and dereferencing was fine.

So what did I have to do to make it work? Surprisingly little!

* Install a rust toolchain on my Linux VM, which is basically curling a script and piping it to the shell.
* Installed `build-essential pkg-config libmilter-dev` packages
* Ran `cargo new milter` to create my project into a new directory called milter
* Pasted the example code from the Rust Milter page into `main.rs` under `src/`
* Added `milter = "0.2.4"` and `libc = "0.2"` as dependencies to Cargo.toml
* Ran `cargo build`, which downloads any non-cached dependencies and also compiles my application
* Ran `cargo run` in the same directory which automatically runs the program without telling it where to find it, not that the source by default runs an internet socket on localhost:3000
* Opened a new shell because cargo run was tying up the console and then edited `/etc/postfix/main.cf` and added exactly what was in the milter code: inet:3000@localhost into the two fields `smtpd_milters` and `non_smtpd_milters`
* Did NOT restart postfix, the change was picked up automatically
* Sent a test email and IT JUST WORKED

## What did I learn?
Linux definitely has that smell of things being more complicated than they need to be. Why do people write 50 page guides on setting up postfix instead of just providing a bash script? Why do I need to create that script when I can barely read variables from the command line without googling it. Don't get me started on trying to build things from source with makefiles, 1000s of warnings that I don't know if it matters and all the rest.

I learned that you can do things in Rust with much less ceremony. It took me all day to setup Postfix on this server, it took me, perhaps 30 minutes to install rust, build a milter and have it already added to postfix!

Enjoy.
