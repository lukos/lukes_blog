---
layout: post
title: The "you're over complicating it" meme
date: '2024-04-07T10:10:00.000+01:00'
author: Luke Briner
---

[This](https://thmsmlr.com/cheap-infra) Thomas Millar article from 2023 appeared on Hacker News today. It is a bit of a meme in our modern day "Cloud" times that clever people enjoy pointing out that everyone who uses the cloud is over-complicating it and the obvious solution is for people to host bare-metal. It is a tiring meme and although I don't think I will add much comment to it that is not already in the Hacker News comments, I find debunking these things good practice for critical thought and for writing practice.

## The Summary
By all means read the article but it goes something like this:
1. You could host a "top 1000" web site using 11MB/s bandwidth on a single bare-metal server/VM
1. Most of you are not top 1000 web sites
1. Therefore if you are using any kind of cloud infra, you are doing it wrong
1. And you are probably paying a premium

My rebuttal is:
1. You mis-represent the top 1000 site's needs
1. You avoid various things like RPO, RTO and availability
1. You do not understand the non-theoretical of many sites that are not top 1000 but still can't use your model
1. Many of us have hybrid systems that use cloud and bare-metal. We do know why we chose that architecture

## The zero-sum fallacy
The article starts from a strange premise. "if you're getting something for a steal, someone is getting screwed". Without defining "steal", it is certainly not a zero-sum world and it is possible to get someone for a reasonable price without paying elsewhere. This slightly strange intro then leads into the point he makes "Amazon makes the majority of its profits from AWS, and yet so many people talk about how the cloud is so cheap...". Do they? I don't remember anyone saying that the cloud is cheap. It can be VERY convenient, especially for early stage companies who don't want the cap-ex or maintenance of a production system until they are sure of their product/market fit but cost alone has never been a reason.

That said, the implication that just because Amazon makes billions of dollars of profit from AWS means we are all over-paying is, again a misunderstanding of how the market works. AWS has over 1M active users according to [this](https://bigsteptech.com/blog/aws-stats-in-2023/) article. Now out of the $500B in revenue (estimated from recent articles), it is making around $20B profit, which is both not very much of its revenue(4%) but when you consider that 10 companies make up a significantly large amount of AWS revenue (Netflix, Twitch, Linked In etc) then how much are we really over-paying? Amazon have a large ability to leverage volume discounts on hardware and are giving jobs to the people who work at those Data Centres, who would otherwise have to be employed by us.

Anyway, I don't think this is a great foundation for the article which is more about complexity and performance than about whether AWS are taking us all for a ride. Also, other cloud providers are available!

## The strawman website
He then choose Business Insider as an example of a "top 1000" web site to base his starting numbers on. 200M visits a month, 2 pages per visit = 400M HTML documents per month.75KB compressed HTML = 30TB of bandwidth.

The thing that is a little weird about this is that you just have to visit Business Insider to see how much of a mis-representation this is. Like a lot of sites, BI is loading things asynchronously over time and uses lazy loading as you scroll down the page so the first impression sizes are not accurate at all. I just clicked through a link there and these are the stats:

* 1873 requests (from one page!)
* 72 of those are documents
* 52MB transferred, uncompressed to 80MB
* Largest load in this page (which is not the page itself) is 249KB
* 16 CSS files, 209 scripts, 6 fonts and 413 images

Now, these memes would normally shy away from using such a site as an example because the purists would say that this site is garbage and should be optimised. That might be true to an extent but I will defend this site and its right to be like this. 

Business Insider (or Insider Inc the company) have 550 employees according to Google and presumably it is a heavy user of advertising which means also a heavy user of metrics to track effectiveness of placement, click-throughs, engagement etc. Now wouldn't it be nice if all of that just came from a single magic library in 10Kb of Javascript but it doesn't. It comes from multiple libraries, some of which might be more useful than others. Some will be large and some smaller. Some make loads of back-end requests asyncronously and God-forbid, some of them have been forgotten about but no-one knows enough to work out whether to remove it, or, like on the Railways, you leave it there forever in case removing it breaks something important.

So if we look at 400M docs per month, even if we could offload all of the static content to a CDN, we are still talking more like 80TB per month, which is 30MB/s, and remember, this is average, not extreme. Although the average might be 2 pages per visit. What about the few thousand people that click into lots of pages. Maybe they have it open all day because they use it in their job. You can't just take 2 pages and 200M people and create a figure that is accurate.

He is also says that 75KB of HTML is a lot, which is laughable. Maybe in his neat niche world where he is still at a stage where he can easily control his sites, this is true but you will struggle to find many pages on the web less than this. Even the Google homepage, the common example of slick content is 76KB compressed so maybe if Thomas is so clever, he should go and work at Google and get their page size down because it is "A LOT".

## The timing fallacy
As many quickly pointed out. As well as the average in the article likely being far below what it actually is, the truth is that traffic is unlikely to come in consistently across the day. Business Insider is largely US material so that means probably (I don't know), a lot of the total traffic will concentrate during the day between EST and Pacific time. What does that mean in reality? Probably much less traffic weekends and overnight meaning what? Maybe 60MB/s AVERAGE and potentially much higher than that if they have just released an important story.

I work at SmartSurvey and our surveys are used by 1000s of respondents across 1000s of customers. You might assume, therefore, that we would have a good spread of traffic but we don't. Some days are quiet (< 1000 on a survey) and other times we have peaks usually > 10K people on a survey. Evenings and nights are quite (we mainly target UK customers) and weekends also that way.

Each of these objections can be brushed away, "yeah, these are only rough figures" but the problem with these types of articles is that each of these inaccurate assumptions are piled on top of one another so the error margin is multiplied for each stage until it isn't just a strawman, it is a strawman made of strawmen!

## The refactoring fallacy
He also makes another common and naive assumption about the way that larger organisations work. "Reduce the HTML size..with a CDN serving your JS, CSS and images...". Again, great for him if that is possible but things are not so straight-forward in the real world. Again, at SmartSurvey, we host a number of apps. Some of these are served by Windows Servers and we also use a CDN and some microservices. Can we move all of our scripts to a CDN? nope. They are generated by various sources and might be built as part of our build process. Is it worth investing the time into pushing them to CDN? Well some already are but in other cases not worth it. Why? Because we already pay for these servers, they are bare-metal so unlike the cloud, I can't hire a cheaper model by moving some of the requests off of them and they are not struggling to keep up. Real business is pragmatic. Spend your time on things that matter.

Could we do it if we were starting from scratch? Possibly but, again, within a year or two most of us are held to a certain amount of restriction by the choices we make, the libraries we use etc (and the things that Marketing tell us we have to embed even though us Developers don't like them).

And we are a 5th the size of Business Insider and much smaller than other companies who run top-1000 web sites so if it is hard or not worth it for us, is it really worth it for them?

## The lazy conclusion
So then he says that newer hardware can easily handle the badly estimated 11MB/s of traffic. Except that's not all the traffic. We already talked about the fact that the average load was probably significantly higher and the traffic probably spikes significantly higher than that.

These articles I read like to pull some headline figure of a well-configured server and quote this is a simplified view of what you are trying to do. What about when your server OS blips or your network has a funny moment and all of those requests back up? What happens when your network card chokes and the errors cascade?

He then makes an even more ridiculous claim that since new servers are not very powerful, "You can server the world from a single box". What utter nonsense. You can't even serve a single netflix movie across the world with a single box, which is why Netflix don't have a single box.

"Why do we need Docker, serverless, horizontal scalability again?". Because you don't understand all of the reasons people use these, clearly, otherwise you wouldn't ask this question.

Let us just take Docker as an example. Yes, it provides the means to microservice deployments and yes, microservices are sometimes used where they are not needed but Docker is not just used for microservices. It is useful for setting up Dev environments since the dependencies are in images, not in the host machine. It makes setting up build agents much quicker for the same reason. Getting a set of working applications/services is much easier than trying to set them all up to run manually. I know because I have done it both ways. I know that pre-containerised workloads was very much a lot of time invested in making my machines work like pets. All of the time wasted setting up new machines and servers because something wasn't documented. We use a microservice for our email sending. It centralises sending from multiple applications into a single place, which is much easier to manage, and it runs in Azure Kubernetes Service for a few hundred pounds per month. Ironically much cheaper than our on-prem services cost to rent. Yes, I did run my own Kubernetes Cluster for a while and that was far too much work and cost much more than AKS in time. I also looked at other options but the simple truth is that this works for us and yes Thomas, we do know there are other ways to do things.

## The missing middle
The article also assumes that business run on a single web application. Easy to optimise that right? Well, no it isn't but easier than the fact that most businesses have 10s, 100s or even 1000s of individual systems and some of them do have to talk to each other. If I put one of my systems onto bare-metal instead of a managed service of some kind, what about all the others? Do the same thing? Do you know what it's like to do Windows and Linux updates each month on bare-metal servers?

As well as the added complexity that isn't really acknowledged in the article at all, the other issue with the complexity is that the time that I could be optimising my simple front-end web site is the time that I am not spending on the other much more difficult but important back-end systems, monitoring, server management etc.

## The funky Edge data
He seems to dive straight into an objection related to the latency at the edge when running a single beefy bare-metal box from a single origin but this feels awkwark because although people do use the cloud for latency reasons, this isn't an objection that he has previously raised. He already talks about using a CDN so depending on this definition of "cloud", which he also didn't define, then he is already using the cloud (the CDN) to get good performance with his bare-metal box. Confused? I was.

Perhaps he is only berating abstracted cloud services, not cloud per-se or maybe CDN providers are good but if they are also cloud providers, they are evil. It is also possible that this article is assuming that people are using SPAs, which although common doesn't really fit into the previous description of Business Insider serving HTML documents. Also, SPAs are usually served by CDN so not a particularly over-engineered solution warranting this article.

The next paragraph says [assuming your JS, CSS and media coming from CDN], "is if you can shave 300ms off your server processing...you've effectively moved your server across the world". In other words, it sounds like the counter of the argument "we use cloud providers because we can use the edge to reduce latency". This another example of a strawman in this article. People use CDNs to reduce the latency of those types of files, perhaps but we all know that there is still a need to make a call (depending on your app) to the origin server. We all know that this latency is important but the CDN is also to reduce the number of concurrent connections that might be made to the origin server and to remove the need to send cookies. All just good practice, not all just with the desire to reach Australia and appear like we are running locally. If I wanted to do that, I would create a more complex architecture so we could have origin servers in Australia. But still, he mentions the CDN and it is still a cloud product, so I am still not entirely clear what he is arguing against.

The rest of this section seems to double-down on the fallacy that we use edge services, like CDNs, to remove all latency and we are obviously wrong. "Nobody ever thinks about the database" is another ridiculous statement. That is mostly what we think about at SmartSurvey because it is the largest "Singlish point of failure" (not quite single because we use a failover but still a good candidate for a global outage).

Even more weirdly, he then defends his unclear argument by saying that you could get most of the benefits you would get from the edge you could get from Cloudfront! Yes, A CDN by a cloud provider (the one he started the article saying was ripping you off) and, YES, a system that uses edge locations to reduce latency.

Then he flips right back to reducing the international 300ms latency by using SQLLite instead of a relational data. Now, the 300ms latency was introduced as the realistic time to get a request across the world, not the time it takes to load a server-rendered page but now we are using it to mean "general slowness of the applcatio". Again: VERY confusing what point he is making. The best I can work out is that "cloud is bad, therefore managed postgres (for example), is bad, therefore just use SQLite" but where did that come from? SQLite hasn't even been mentioned before and all of a sudden, this is a solution to people using cloud? Why not just suggesta locally managed database instance. On both our on-prem SQL Server and our Cloud VM hosted postgresql cluster, we get single-digit mS access times so not really related to anything significant of the 300ms application latency or anything related to global latency.

## Ah yes, the cost
Cost is a very common red-herring when people write this kind of meme posts. To people who lack experience, something like $500/month for an AKS cluster is extortionate. You could rent a VM for only $10/month and even if you wanted 6 of them to make a redundant cluster, it would only be $60/month you fools!

When you turn over £4M/year, an extra $400+ per month is nothing. Literally a rounding error on the accounts. Sure, we could save $400/month at what cost? My time is probably worth $1000/day so who is being the fool? I did try once to manage a Rancher cluster and it was a nightmare. The docs were all over the place since things changed between versions, there were multiple types of cluster but it was never clear which was which and upgrading a major verison of Kubernetes, which is point-and-click in AKS required me to get our Data Centre provider to write some scripts which I had to run manually on the VMs. Of course it is doable but again, it becomes a pet. I probably spent more than $400 month just keeping the thing running not to mentioned regular node draining for Linux updates. Sure, maybe only fools use microservices (or Kubernetes) but I prefer Kubernetes to restart failed pods than being woken at 2 in the morning because our email system has stopped working.

Maybe we should have a 24/7 support team to take care of that? Yep, that's at least another $100K/year and that's on top of the recruitment time, replacing people who leave or paying the premiums that you don't want to pay AWS to a Contract Support Business instead?

In a familiar vein, Thomas then uses Hetzner.com (a cloud provider) to demonstrate why AWS is expensive. Is this whole article just a rant about AWS? Or are you trying to dress it up like a cleverly reasoned article with its very selective choice of comparisons? So the providers charge different amounts? Doesn't sound unreasonable. Every business in the world decides on their pricing model. AWS gives away free tiers and then charges. Some others do, some don't. SmartSurvey has unlimited responses on all business and enterprise plans, Survey Monkey, instead, gives you all the features for free and charges for responses. That's OK, it's up to us and it's up to them. If you don't want to use AWS because they are a rip-off, there are others you can choose but you pay your money and take your choice. Do you want AWSs more data centre locations? Or OVH's EU based business or Hetzner's prices? Great.

He also mentions "edge cloud vendors" whatever they are. Vercel charges $200/TB once you pass the free tier. Great, let them charge what they want. if it doesn't work for you, don't use them but implying that people who choose the cloud are stupid because some companies charge more than you think is reasonable is disingenuous at best and ignorant at worst.

## The reality of the situation
Never trust someone who makes a statement about the "reality" when talking about such a large topic as this. The reality is simply that providers provide services that work for some people at a price they are prepared to pay and not for others. That is fine.

Do some people over-engineer? Of course they do, just like some people always want the latest version of Vue JS, it doesn't follow though that everyone who uses the latest version of Vue JS is over-engineering or stupid or following a band wagon or whatever.

Saying that you can probably run on a single server is again, naive at best, and a gross simplifaction at worst. If you want to run your system on a single server. Great. Do it and good luck but please don't tell me that I can or should.

"Use SQLite locally on the box, you don't need a managed database". These are not the only options, this is the false dilemma fallacy. You could use SQL Server or postgresql on the same box, or other boxes nearby, or in your own data centre. You could run it on VMs in a cloud provider or something else. This statement is neither necessary for the overall post (whatever that is) and is not even true. He also throws in, "Use Litestream to back it up". Why? Are you trying to sell Litestream? Why only that option. Why not, "make sure it's backed up"?

"Your CI can just SCP your code to the box". How do you know? That's not how .Net Framework works. Nginx supports zero-downtime. That's great but not everyone uses Nginx or can use Nginx or wants to use Nginx.

"Don’t bother with docker and virtualization and all that nonsense". Already mentioned this. The author doesn't seem to understand all of the reasons why people use docker and virtualization.

"They say you'll need to scale..." Who're they? That doesn't seem relevant at all to this discussion. I can can scale with on-prem servers and with cloud services, just another irrelevant sound-bite thrown in to make the article sound more authoritative.

"If you really care about latency, throw a server in Germany, and one in California, route your writes to your primary, use your local read replica for reads.". So what? Do what everyone is already doing before you told them all to use a Single Server and SQLite?

## Conclusion
Like all of these other similar types of articles it is patently nonsense. The calculations are fag-packet at best and potentially far, far, off the actual reality but we already know that because those of us who run large sites already know that. Contrary to popular opinion, a lot of us who make these decisions do understand the trade-offs and although our decisions are not always perfect and maybe we sometimes regret our decisions, I am happy that we run on cloud services as well as on-prem servers. I also look forward to the day that we can run more applications on this "Docker nonsense" which significantly reduces the amount of my time I have to run these systems and even at $50K/year, it is still significantly cheaper than employing people to do all the stuff that Azure or AWS do for me, even if they are charging a premium for it.

I have run small VMs for scenarios where it works, one of them runs a local copy of MySQL and it's great but it wouldn't work for what we do at SmartSurvey. Maybe one day, we will get rich enough that we could save a few million by bringing things in-house but I doubt it. These things do repeat every 10 years or so and I am waiting for the first article describing how, "We brought the cloud in-house and it was the worst decision ever", not because it always is but because some things work for some people and not others.

As with every engineering decision ever, your mileage may vary. It always did and it always will.