---
layout: post
title: "The cost of cloud"
date: 2016-07-30 13:24:28 -0400
categories: []
author: 'Marco Ceppi'
comments: true
---

The infrastructure has been humming along so far, and except for the occasional blip during a code release it's become clear we've done a stellar job of tuning the infrastructure as well as the code using [data to help drive optimizations]({% post_url 2016-07-16-insights-with-datadog %}). So, it's getting close to the end of month, a time when bills are commonly due. Peaking into the Amazon account billing board a surprise awaited: the cost of our popularity so far.

![Imgur](http://i.imgur.com/zLA8cxi.png)

Ouch. There's a common saying that the cloud is expensive, so I steeled myself for the bill. However, I wasn't prepared for the cost of a CDN. A CDN which we celebrated not too long ago. With every decision there's a trade-off, with being "on rails" and super focused this past month, these pros/cons tradeoffs increase in risk. This is why data-driven operations is so important. We're going to have to eat this cost and this choice, something we should have been monitoring closer - sooner. So we've identified two problems. The first, we're paying way too much for a CDN. The second, we're serving too much data.

![Imgur](http://i.imgur.com/cYILbR3.png)

The first problem is a little harder to fix and requires engineering time. The second problem, however, is something we can aciton upon today. After taking a look at our throughput I turned to the internet in order to quickly find a CDN server which could offer the infrastrcture for the amount of data we're pushing out while still being cost effective. I ended up landing on [CDN Calculator](http://www.cdncalc.com/) which, while rudimentary, it was able to help scope my focus. In following the trend of the past month we're looking for breathing room, another $3,000+ bill will severely prohibit The Silph Road going forward. After entering all the data that we've collected for the past two weeks while on CloudFront, [CDNify](http://cdnify.com) seemed like the best intermediate solution.

![First site when you Google cheap CDN](http://i.imgur.com/HBVE1n6.png)

To quickly answer why I chose CDNify over JoDiHost, I briefly looked at both and at the end of the day, CDNify seemd to be a pure CDN company. This means they'll have a much more focused user experience and provide the most insights into our data. This is something that Amazon lacks in CloudFront. The major difference between this CDN and AWS CloudFront: AWS CloudFront is backed by S3, CDNify simply caches from our origin. This means in order to warm the cache we have to let loose a bunch of traffic into the loadbalancer. I knew this would spike things, but I didn't realize fully the implication until we made the switch in the site configuration.

![whoops, that's a lot of cache warming](http://i.imgur.com/Aw3WUhZ.png)

This isn't really a bad thing, we can handle a flux like this, but it does put undue strain on the servers. Servers, which we'll be scaling down very soon to continue saving costs. As a result, we can't have assets being fetched from our servers every 24 hours (or however long we make the TTL) - same thing when we update assets on the site. This is where I really like CDNify's interface. I was able to, very quickly, repoint the origin from thesilphroad.com to our S3 bucket which was originally backing CloudFront! This means we still offload our traffic, and the amount of traffic hitting S3 is very low and predicatable, which we can plan for.

Over the next few days we plan to work on making image sprites of all the multi-image assets. This means instead of 151 pokemon images that have to be loaded 151 times per page view, it's a single PNGcrush'd image that's distributed once from a CDN cache. This will cut down on the amount of bandwidth used as well as the number of requests to the CDN - both things that we're charged for.
