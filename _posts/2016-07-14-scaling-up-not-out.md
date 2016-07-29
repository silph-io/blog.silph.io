---
layout: post
title:  "Scaling out, not up"
date:   2016-07-14 14:18:23 -0500
categories: []
comments: true
author: Marco Ceppi
---

Joining The Silph Road meant a shift in architecture. The platform's hosted on Amazon's Cloud where the application, the Apache web server, and MySQL lived on a single machine. This isn't a bad design, by any means, and is quite common for "LAMP" (Linux, Apache, MySQL, PHP). There is, however, trade-offs to this design. The biggest  of which is the limitation of scale, which is what we hit during the first code launch.

When traffic hits, the load on a single server becomes clear. Apache, PHP, and MySQL are all fighting for machine resources so the only way forward in this model is up.

![Imgur](http://i.imgur.com/wXgq8oN.png)

This was the Reddit post that caught my attention. There comes a point where you can't physically get a bigger sized machine (or VM in a cloud). At that point, you need to start considering a scale-out architecture as a path to handling more requests. The general principle of scale out: instead of one machine doing everything, you have many smaller machines processing requests. These machines typically get grouped into multiple tiers. These tiers each function towards a single goal. We separated each concern and created two tiers: A database tier and application tier. Since we have more than one machine in the application tier we need a way to route requests properly. To do so we'll use HAProxy, a load balancer, designed to route and balance traffic to one or more machines.

![Imgur](http://i.imgur.com/i0h0NKO.png)

Additional to this separation, there were also a few product changes. Where before the architecture was Amazon Linux, Apache2, MySQL, and PHP; the new architecture is Ubuntu, NGINX, MariaDB, PHP-FPM, PHP. Aside from moving to a more updated and secure Linux, the other choices were more personal. I have much more experience with NGINX and PHP-FPM than Apache and mod-php. MariaDB has tested to be a more performant replacement for MySQL.

All this sounds great, but how we put this in place isn't a trivial exercise. With ample time tools can be evaluated and experimented with. However, we’re scaling under load and under fire and can't afford this grace period. This is why we're using Charms and Juju to drive our infrastructure.

Charms themselves are the encapsulation of the operational logic for each piece of software. We don't have time to become experts at scale for each component: MySQL, HAProxy, NGINX, etc. What charms provide is a curated, cloud-optimized, cloud-agnostic, and reusable set of expertise. This means we're not locked into choices we make now and can scale/change components, or clouds, at any time.
