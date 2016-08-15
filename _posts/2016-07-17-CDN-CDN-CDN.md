---
layout: post
title:  "CDN, CDN, CDN!"
date:   2016-07-18 19:22:55 -0400
categories: []
comments: true
author: Marco Ceppi
---

Based on the [data we're getting from Datadog]({% post_url 2016-07-16-insights-with-datadog %}), it's clear disk utilization is the biggest boon in our deployment. One thing that hits the disk a lot is the loading of assets: images, javascript, css, etc from the servers to you, the user. On average each page has nearly 200 assets to load (If you can imagine, each pokemon image means a minimum of 151 images :wink:). In order to prepare for invites going out, we wanted to make sure we could handle the load without having to scale more. One way to help offload stress of each image, css, and javascript request hitting the server is to put these static assets into a CDN.

Content Delivery Network, or CDN, is a means to distribute files from multiple locaitons around the world. If you imagine the following diagram is how things work today:

![Imgur](http://i.imgur.com/R4qVYpW.png)

As you can imagine, if you're on the eastern seaboard of the United States, getting images to load from the site is a lot faster than if you connected from India. This is because there's less infrastructure you have to travel through from New York to Washington DC than from India. What a CDN offers is the ability to place our static content closer to you! Most CDNs have multiple POPs (points of presence) where they replicate the content to those nodes. What that means is, when you visit a website, instead of going all the way to the Washington, DC area, you're actually getting the content from a POP that's closest to you. Here's a simplified view of what a common CDN POP map looks like:

![Imgur](http://i.imgur.com/vJISuz4.png)

Now, if you're in India, the content will load just as fast as if you were accessing it from New York. For our CDN I decided to stick with Amazon and use their "CloudFront" service. To enable this was pretty easy, I just uploaded the content to an S3 bucket then told CloudFront to syndicate that bucket. The hardest part was getting the content to serve from that address. Typically we would change these URLs in the application, but we didn't quite have time to hunt this down, build this in, then deploy it. So the change as actually made at the loadbalancer level by adding the following to the frontend definition:

```
    acl is_img path_beg -i /img
    http-request redirect code 301 location https://d24wnu4kg3mkno.cloudfront.net%[capture.req.uri] if is_img
```

While simple, this change basically says if any request coming into the loadbalancer starts with `/img` then we redirect that to the CDN URL instead. This means [thesilphroad.com/img/pokemon/animated/blastoise.gif](https://thesil
phroad.com/img/pokemon/animated/blastoise.gif) gets redirected to [d24wnu4kg3mkno.cloudfront.net/img/pokemon/animated/blastoise.gif](https://d24wnu4kg3mkno.cloudfront.net/img/pokemon/animated/blastoise.gif). That URL will point to a different POP depending on where you're physically located. In the coming weeks we'll make this URL configurable in the codebase so that instead of having to go ALL the way to DC just to get repointed somewhere else, you'll go straight to that URL from the start. This will help lower the stress on the loadbalancer and improve performance for everyone visiting the site.

![Imgur](http://i.imgur.com/sj0YfCK.png)

This is what the cut over looks like. Around 1:15 is when we switched over to the CDN, backend traffic is way down. However, frontend sessions have jumped up. This is because we're handling the rewrite at the to the CDN. Either way, the goal is accomplished by lowering the stress on the servers disks and increase the sites responsiveness by having assets load faster for users.
