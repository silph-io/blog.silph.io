---
layout: post
title:  "Moving clouds"
date:   2016-09-10 23:17:16 -0400
categories: []
comments: true
author: Marco Ceppi
---

We're just shy of two months running on the new scale-out infrastructure and a common thread throughout those two months is the cost of running in the cloud. We first trimmed down cloud costs by [moving to a CDN]({% post_url 2016-07-17-CDN-CDN-CDN %}). We then followed up by moving to an [even cheaper CDN]({% post_url 2016-07-30-the-cost-of-cloud %}). Finally, resting where we are today, with [MaxCDN](https://maxcdn.com) which has actually been a good balance of cost and quality. However, this is a community run website, and to make sure we keep the lights on - so to speak - we need to make sure we're getting the best deal for everything we do.

To date, The Silph Road runs on Amazon's Web Services. While I've expressed some distain towards them, they are arguably the largest cloud vendor. As a result, AWS comes with a lot of features, is quite robust, and well trusted.

In our [last post-mortem]({% post_url 2016-08-30-postmortem-tuning-app-and-database %}) one of the questions in the comments was why we don't leverage more of these features. Our architecture is rather simple where we just consume a set of machines, IP addresses, and disks. Amazon alone has over 50 additional services, from databases as a service, dns as a service, object store as a service, containers, email, queue messaging, the list goes on. These are quite enticing. You get an endpoint + credentails and don't have to manage anymore infrastrucutre. However, it comes at a cost - and the most expensive of those costs is vendor lock-in.

Last months (August 2016) Amazon bill was $1,200. For some that's a lot, for others not so much. Regardless, it's more than we want to pay a month for _just_ cloud instances. When you start adding up the cost of the CDN, and other services we depend on, we reach a point where keeping the lights on is a real concern. As a small team, we must be agile and swift. To be agile we try to avoid cases where we're married to any one platform - or tool.

When it comes to public clouds, there's actually quite a bit of choice. In addition to Amazon, there's clouds from other industry titans: Micosoft's Azure Cloud and Google's Compute Engine, but also other budding clouds: Rackspace, OVH, Joyent, CloudSigma, the list goes on. After spending quite a bit of time mapping out projected costs of the infrastructure, Google's GCE seemed to offer the best performance for price point. Now that we have a cloud chosen, it was time to do the migration.

Normally, when it comes to migraitons of this scale a maintenance window is usually announced, then a few hours are spent moving everything over. We're not a Fortune 500 company or some large platform, and maintenance windows/scheduled downtimes are so old school. Here's how we pulled off the migration with less than 2 mins of interrupted service.

The first step was to get the new cloud instances up and running. To do that we ran the following:

```
juju bootstrap silph.io-prod1 google/us-east1
juju deploy -n 4 cs:~silph.io/silph-web --constraints instance-type=n1-standard-2
juju deploy mariadb --constraints instance-type=n1-standard-2
juju deploy redis
juju deploy haproxy
```

So to lay out what we've done:

 - Create a Juju controller on Google's US-East1 region called `silph.io-prod1` (Amazon's controller is called `silph.io-prod0`)
 - Deploy four units of the silph-web charm. This charm is all the operational knowledge of how to install, configure, and manage the Silph Road web app
   - We tell Juju we want these to be on the [`n1-standard-2` instance types](https://cloud.google.com/compute/docs/machine-types#standard_machine_types).
 - Deploy MariaDB, Redis, and HAProxy from the charm store. These are charms created and maintained by the upstream vendors or upstream communities

Now we've got seven machines running, four are our web application and three are split between database, cache, and load balancer. Right now they're all running but unconfigured. The web application doesn't know where the database or cache servers are and the loadbalancer doesn't know where the web application is. In the Juju Charms, the relations are encapsulated as code.

![Models](http://i.imgur.com/RRR3RVO.png)

So the silph-web charm says it consumes something called "mysql" and "redis", and provides something called "http". Likewise, redis and mariadb provide "redis" and "mysql" respectively, and haproxy consumes "http". This mapping allows for an explicit exchange of information, like connection details, without me or anyone else having to interfere. To make these connections we just issue the folowing:

```
juju add-relation silph-web mariadb
juju add-relation silph-web haproxy
juju add-relation silph-web redis
```


We also need to bolt on our monitoring!

```
juju deploy cs:~silph.io/datadog
juju add-relation datadog haproxy
juju add-relation datadog redis
juju add-relation datadog mariadb
```

We then need to pass in some configuration, things like our access key for Datadog and the SSH private key to access our code repositories

```
juju config silph-web ssh-key=$(cat $HOME/.ssh/silph-repo)
juju config datadog api-key="my-secret-api-key"
```

It'll take a few mins for the environment to go stable, this is because distributed systems will always take a varying amount of time to converge. Since we're using Juju and Charms, I don't have to worry about the ordering of events. In fact I could have run any of the aforementioned commands in any order and gotten the same results. Now that everything is idle and converged we can inspect the status with `juju status`:

![Imgur](http://i.imgur.com/RENsY6G.png)

Now it's time to do a little razzle dazzle. To prepare we need to do the following:

 - Set DNS time to live (TTL) to as low as humanly possible
   - This will allow us to change the IP address for thesilphroad.com and hopefully have people's browsers get the new address quickly

We lowered the TTL ahead of the move to make sure everyone had the new lowered value. Now the following will happen all in the same 1 minute window:

 - Change IP address in nameserver to new HAProxy IP address
 - Take a snapshot of the database
 - Disable the database server on old enviornment
 - Restore snapshot of database to new database server
 - Update old loadbalancer to send all requests to the new loadbalancer

As you can see, it's a lot of moving parts! We disable the old database server on the old host after the dump to make sure no new data is written. The downside is while the database is being restored to the new server people will get an "Oops" message. It only takes about 45-60 seconds to move the database snapshot to the new server. Once it's moved over we can't count on DNS being updated, so we set the load balancer to redirect all requests to the new server. The result is a ~90 second window where the site is unavailable.

While I'm quite happy with the result, I've got plans on how we can actually do a full switch over with zero down time. As we've got more migrations coming for the infrastructure I'll cover those plans in another post.

For now we're on Google and so far things seem to be going pretty well!
