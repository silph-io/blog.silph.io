---
layout: post
title:  "Saving user sessions (and our disks)"
date:   2016-07-22 21:45:16 -0400
categories: []
author: 'Marco Ceppi'
comments: true
---

One side effect of the move to [multiple, scaled out, architecture]({% post_url 2016-07-14-scaling-up-not-out %}) is how user sessions are saved. Whenever you log into the server you get back a cookie with you session identifier. This tells the server who you are without you having to log in everytime. On the server, it stores a copy of that identifier to validate that you are who you are and you still have a valid login. Before the move, when everything was on a single machine it was easy to validate these sessions. You always visited the same machine and your server-side session was there.

Since our move, when you access the site you may actually hit one of five machines. If you login to server 1 and on the next page view end up on server 3, the code doesn't have your copy of the server-side session and you're forced to log in again!

![Imgur](http://i.imgur.com/i0h0NKO.png)

When we set up the loadbalancer we used an additional cookie configured at the loadbalancer. Whenever you first hit the site, if the cookie isn't set, the loadbalancer picks one of the five servers to send you and sets the cookie value to that server. Going forward, every request you make will always end up at that server. However, the cookie is designed to expire frequently, why? Well we move servers around all the time.

```
backend silph-app
    balance leastconn
    cookie SERV insert

    server silph-road-0-80 10.16.237.223:80 maxconn 350 cookie s0 check
    server silph-road-1-80 10.155.40.82:80 maxconn 350 cookie s1 check
    server silph-road-2-80 10.233.108.106:80 maxconn 350 cookie s2 check
    server silph-road-3-80 10.237.209.237:80 maxconn 350 cookie s3 check
    server silph-road-4-80 10.233.128.131:80 maxconn 350 cookie s4 check
```

Ultimately, it's just not a solution that will scale. Something else we noticed, as we got a jump in traffic, the performance of the site degraded again. This was the same [pattern of degredation]({% post_url 2016-07-16-insights-with-datadog %}) we say before moving to a CDN: PHP processes ballooning out of control on IO wait. On waiting for disks!

When we started drilling in, one thing we noticed was a lot of time spent getting a file written to disk. Ah! One thing that wasn't accounted for, the more people visiting the site, the more session files that get created. This locks the disk up just as bad as before when we were only reading from the disk for assets. In addition, we've been getting a lot of reports of "login" loops. Where after authenticating a user has to authenticate again several more times before it "sticks". This presented us with the ability to kill two birds with one stone: offloading sessions from the disk much like we did assets with CDN. To do this, we'll need a datastore that can handle the amount of traffic without adding more overhead.

For this, Redis was the obvious option. We've looked at memcached and I've used it in the past but when we tried to make the switch during the original setup, it ended up being a huge bottle neck. Redis is a very VERY fast, in memory cache. By being in memory it never writes to disk so it won't incur the overheard that we see today. It's also very lightweight handling many transactions without stressing.

![Imgur](http://i.imgur.com/YTPFzQB.png)

The switch to Redis as a centralized cache for sessions has done two things. First, no more session drops! If you don't visit the site in two days from the same device, you'll have to log in again, but that's because we don't want sessions staying around too long. Secondly, the average response time from the backend has gone from over 500ms on average to under 15ms! This means we can process more requests a second without stressing the servers of having to scale up.

