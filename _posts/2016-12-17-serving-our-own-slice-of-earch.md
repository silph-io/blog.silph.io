---
layout: post
title:  "Serving our own slice of earth"
date:   2016-12-19 06:58:16 -0500
categories: []
comments: true
author: Marco Ceppi
---

**This is part one of two**

Things have been pretty quiet on the Silph Road Ops. After ironing a lot of our [scale issues]({% post_url 2016-07-14-scaling-up-not-out %}) and [monitoring snafus]({% post_url 2016-07-16-insights-with-datadog %}) the servers basically run themselves. Dronpes, Moots, and the team have kept on updating, upgrading, and releasing the Silph Road. The goal of this blog, and Silph IO, is to make sure the site stays up, stays healthy, and is cost effective to run. Today we're announcing another step in making sure both goals are met.

> Due to some, frankly, aggressive and deceitful sales tactics; Mapbox went from our savior [...] to potentially a nail in our coffin

When The Silph Road first launched, back in July and August and before the redesign with nest atlas, we used Google Maps for sightings. The invite system was largely due to connection limits with the free API key. While there are certainly ways around this, we believe in respecting TOS and licenses for all aspects of our stack. As we moved into August, Dronpes replaced Google Maps with Mapbox. Mapbox is one of the primary, third party, basemap providers. Effectively, they're the underlying images that we overlay all the nest pins on. Due to some, frankly, aggressive and deceitful sales tactics; Mapbox went from our savior from invites to potentially a nail in our coffin. For specifics, Dronpes covers this part in [more details](https://www.reddit.com/r/TheSilphRoad/comments/5jb99f/saving_the_nest_atlas_our_expenses_enterprise/).

Needless to say, just as when [Amazon's cost]({% post_url 2016-07-30-the-cost-of-cloud %}) became untenable it was clear that Mapbox would not fit into our stack. Unfortunately, there seems to be a bit of a monopoly on basemaps. The number of companies providing this service are in the tens, the prices go from "developer account" to "call us", and every company we shopped with couldn't give us a price that works for our budget.

Before I go a little too far. I want to dive into the idea of basemaps a bit more and help explain some context. Most people are familiar with basemaps, though they don't realize it. When you load up Google or Apple Maps, you're presented with a canvas of a digitalized rendering of your location. Streets, building outlines, points of interest, etc. These are types basemaps. It's the lowest level of a map display, much like a paper map of days gone by. From that basemap, Google, Bing, The Silph Road applies layers. Layers are anything that extends that basemap with supplimentary data. Driving directions with a highlighted route is one example, traffic reports another, or pins showing nest locations. I mentioned previously basemap providers are in the 10s. Data sources for basemaps is even less. For the most part, there really only exists ONE public source of basemap data. [OpenStreetMap](http://openstreetmap.com/about).

OpenStreetMap (OSM) is much like The Silph Roads nest atlas, but for geographical data. Thousands of contributors scour their respective areas, collecting data about roads, parks, buildings, intersections, rivers, mountains, and any other geospatial data. These contributors update OpenStreetMaps database much like how you update/submit nests, or how you could edit a Wikipedia article. It's an incredible effort, and a platform I'm a huge advocate for. Larger companies, like Google, Apple, etc keep their basemap data private. Smaller companies, like Mapbox, reuse OSMs data. This is possible because all contributions to OSM are licensed under the creative commons (CC-BY-SA). Which states, in brief:

> Share — copy and redistribute the material in any medium or format, Adapt — remix, transform, and build upon the material for any purpose, even commercially.

This is interesting for many reasons, but the main reason is because while Mapbox is repackaging and reselling the generated images from this data, we could in effect, use the same data Mapbox has to produce our own tiles. In less than a week that's exactly what we did. What follows is a much more technical breakdown of the process and initial cost analysis. But to summarize here for those not quite interested in the details, Mapbox final quote was $180,000/year while rolling our own infrastructure will cost less than $10,000/year (or $15,000/mo vs $987/mo!!).

# Technical Details

Before diving too deep into this post, I'm going to introduce a lot of OSM and basemap terminology. A lot of these concepts were new to me so I'd like to quickly define them:

- **[Element](https://wiki.openstreetmap.org/wiki/Elements)**: the basic components of OpenStreetMap's conceptual data model of the physical world consisting of nodes, ways, and relations
- **[Node](https://wiki.openstreetmap.org/wiki/Nodes)**: represents a specific point on the earth's surface defined by its latitude and longitude
- **[Way](https://wiki.openstreetmap.org/wiki/Ways)**: an ordered list of between 2 and 2,000 nodes that define a polyline, typically used to represent linear features such as rivers or roads
- **[Relation](https://wiki.openstreetmap.org/wiki/Relations)**: a multi-purpose data structure that documents a relationship between two or more data elements (nodes, ways, and/or other relations)
- **[Tile](https://wiki.openstreetmap.org/wiki/Tiles)**: square bitmap graphics displayed in a grid arrangement to show a map, typically 256x256 pixel
- **[Meta Tile](https://wiki.openstreetmap.org/wiki/Meta_tiles)**: a server side construct of a square area consisting of 8x8 normal __tiles__ (64 normal tiles) covering a square area of map which is 2048x2048 pixels

After quite a bit of research earlier last week, it became clear that we'll need a lot of CPU power and a lot of disk space. Thankfully, disk space these days is cheap. CPU time isn't as expensive as it's been in the past, but it would clearly be where a lot of investment went. Today, we run the majority of The Silph Road on Google's Cloud Engine, for this I wanted to experiment with Amazon again. I know it costs more, but it'll be a good marker of maximum costs and provides a baseline for performance comparisons. As we wrap up this initial experiment we'll likely consolidate this elsewhere.

We started originally with a r3.xlarge (4vcpu / 30.5GB), this was changed out for an r3.4xlarge (16vcpu / 122GB). I know I just went into how this was a CPU intensive process, but the first half is very VERY memory intensive. Once booted we attached two disks, one was 50GB io2 and an 800GB io2. These are ext4 partitioned with the labels `mapdb` and `mapdata`.

From here we installed a slew of library packages, PostgreSQL 9.5, mapnik, mod_tile, Apache2, osm2pgsql. The end goal is to produce an architecture where all elements of OSM are stored in PostgreSQL and tiles are rendered from that data. This is what we're striving for as an architecture:

![diagram](http://i.imgur.com/0525bGY.png)

1. A request for a tile is made by the client. If the CDN has cached this tile it'll return immediately.
2. CDN requests the tile from the tile server, caching the response.
3. mod_tile is triggered, if the tile exists on disk (tile_storage) it will be returned.
4. rendered is triggered, querying PostgreSQL for the elements, creating the tile image, and storing it on disk (tile_storage) mod_tile then returns the generated image.

To start, we needed to load a copy of the OSM. The OSM project publishes [weekly dumps of their datasource](http://planet.openstreetmap.org/), licensed under the [Open Data Commons Open Database License](http://opendatacommons.org/licenses/odbl/summary/). They use a [PBF format](http://wiki.openstreetmap.org/wiki/PBF_Format), which compresses the data down to 34GB. This file is loaded into PostgreSQL using `osm2pgsql`, which looks something like this:

```
osm2pgsql --slim -d gis -C 24000 --number-processes 4 /media/mapdb/planet-latest.osm.pbf
```

This process takes, well, a while. It took 2.5 days on an r3.xlarge. On an r3.4xlarge it was down to 18 hours. osm2pgsql is quite clever in how it does it's data import. First iterating over the raw data, importing it into temporary tables. Then using the PostgreSQL engine to compute, reduce, and index the data. This requires large amounts of CPU and Memory, which is why we used the r3.4xlarge. Once the import and processing was done, we moved the server to a c4.2xlarge. The final stats, for the planet, as of December 14th 2016 is: 3,636,347,854 nodes, 380,795,405 ways, and 4,626,332 points. Or, 710GB of PostgreSQL storage.

So, we have the data in a format that we can use to generate tiles! For this, we need `renderd`. Renderd is a daemon service that runs in concert with mod_tile. mod_tile is an Apache module that interprets requests for tiles and checks for an existing Meta tile on disk. If they don't, it'll pass the request to renderd. renderd decouples the image create process from the Apache forks, allowing for Apache and mod_tile to process a lot more requests.

renderd, once given a zoom level, xtile, and ytile, will query the PostgreSQL database for the attributes in each of these tiles and compile a Meta tile using an XML/CSS combination to draw the elements. The result is a URL like `https://tileserver/18/236773/161169.png` to the following

![a tile!](http://maps.silph.io/pgo/18/236773/161169.png)

Now that we're getting tiles, we pre-rendered zoom levels 0 through 14. This takes about 62GB of space for the meta tiles. For this we use render_list, which simulates requests for tiles with given parameters. So `-a`ll tiles maximum zoom 14, minimum zoom 0, 50 threads, and maximum load of 22 on the machine. Or:

```
render_list -m default -a -z 0 -Z 14 -n 50 -l 22
```

Okay, we've got the tools, we've got a cache of tiles, we have a rendering server, and we've got all the map data we need. Next was a stress test. As we learned [in the past]({% post_url 2016-07-14-scaling-up-not-out %}), just having a single server regardless of how beefy it is, will eventually succumb to too much load. There's a ton of ways to go about stress testing, for the quickest route we decided to roll the single server into production for a short period of time. During that time we were able to gather enough information from load on the servers, number of tiles requested in a minute, to what PostgreSQl does under full force. Based on that it was clear we needed to increase our rendering capacity. Thankfully, renderd scales!

There's not a lot (okay, none so far) in the way of documentation for this. After a bit of trial and error, it turns out to be pretty simple. To faciliate this, I first mounted an elastic file system. For Amazon this is EFS, but CephFS, NFS, or really any other software that presents a filesystem over the network will do. Then I setup two additional servers with mod_tile, apache, everything with the exception of PostgreSQL. This didn't turn out exactly how I expected. In the end, removing Apache and setting up renderd to use the other servers as "slaves" meant less overhead and better management.

We've scaled this out for the initial bump, with five render slaves running on `c4.xlarge` amazon instances we've started performing the roll out. Dronpes has implemented a staged rollout for the atlas, which went live Sunday night. Over the next few days you may end up seeing a slightly different map and there's a chance you may hit some oddities.

![map-compare](http://i.imgur.com/wkbphfC.png)

If you come across black/grey squares or other issues, try zooming out one level and zoom in. You've hit a bit of a peak moment in the servers' map generation not previously accessed.

I understand this post is a bit light on the actual how to. The following post will dive deeper into this and our plans around basemap generation. For now we're happy with the progress, as we run this in full we'll report back to the community on cost and improvements.

We had less than a week to build a tile server and basemap system, this is by far not the last iteration. In the coming days we'll be stabilizing the platform, updating tile styles, and making sure our tiles are as up to date as the upstream OSM datasource.
