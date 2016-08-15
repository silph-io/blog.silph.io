---
layout: post
title:  "Creating and reusing operational knowledge"
date:   2016-07-15 22:10:16 -0400
categories: []
comments: true
author: Marco Ceppi
---

The first thing after joining was to start [encapsulating the deployment of Silph Road as repeatable code]({% post_url 2016-07-14-scaling-up-not-out %}). To maximize flexibility, we use charms written in Python to avoid DSL lockin and allow for more contributions. Since we're using NGINX and PHP-FPM, we leverage existing charm layers for these to just focus on what it takes to deploy the application and not reinvent every piece of solution. I know what follows may seem like a lot, after all it's just shy of 100 lines of Python code and some meta data, but this is the basis for all the magic that happens in the deployment. It's how we can scale elastically while still only being volunteers in the project of this size.

Before even getting that far though, I needed to figure out exactly how to setup the site. For this, I used LXD. For those not familiar, LXD gives you the equivalant of a virtual machine but without the VM overhead. It's very similar to the primatives that you'd get from another hypervisor - like VirtualBox - but they are very lightweight. It's also a great way to emulate the production environment without having to pay for more machines in a cloud. I downloaded the code to `~/Projects/silph/web-app` and launched a few containers:

```
lxc launch ubuntu:16.04 silph-web
lxc config device add silph-web source-code disk source=/home/marco/Projects/silph path=/srv/silph
```

This creates a fresh Ubuntu 16.04 cloud images and then maps my code directory into the container under `/srv/silph`. Now you can view the containers using `lxc list`, access the container with either `lxc exec silph-web bash` or just `ssh ubuntu@[container-ip]`. Once in I was able to install NGINX, php-fpm, then spent a few hours installing dependancies, tweaking configuration files, and validating the setup.

The next bits are pretty technical, if you're adverse to code you might want to skip the next few paragraphs. Once you have a plan for execution it time to dig in. Charms are pretty straight forward and typically have the following directory structure:

```bash
.
├── config.yaml            # configuration that can be tweaked at run time
├── icon.svg               # icon to make things look pretty
├── layer.yaml             # build-time definitions and configuration
├── metadata.yaml          # describes charm meta and connection endpoints
└── reactive
    └── silph_road.py      # deployment logic
```

We'll start with the `metadata.yaml`, this file is mostly just human-readable fluff, but it does also describe how this charm connects with other deployed components. This is important as we'll need to integrate with a MySQL server as well as a load balancer.

```
name: silph-road
summary: Deploy thesilphroad.com
maintainer: Marco Ceppi <marco@ceppi.net>
description: |
  Silph Road web app
provides:
  website:
    interface: http
requires:
  database:
    interface: mysql
```

Skipping the top half, the key takeaways are the `requires` and `provides` sections. This describes what things The Silph Road consumes and provides, mainly being it consumes a MySQL compatible connection and provides an HTTP compatible endpoint. By declaring these two bits Juju will be able to connect this to the [MariaDB](https://jujucharms.com/mariadb/) and [HAProxy](https://jujucharms.com/haproxy/) charms.

Just declaring this bit of data isn't everything though, we'll also need to declare the other layers we need compiled into the charm. There are two types of layers: a charm layer and an interface layer. Since we're declaring support for both mysql and http we'll need to include those interface layers as well as php-fpm and nginx charm layers as we build on top of that technology. Here's the following `layer.yaml` file:

```yaml
includes:
 - 'layer:basic'
 - 'layer:nginx'
 - 'layer:php-fpm'
 - 'interface:mysql'
 - 'interface:http'
options:
  php-fpm:
    packages: [gd, mysql, imagick, mcrypt, zip, gd, geoip]
```

Breaking this down further, we declare a set of layers we need to build upon in order to get the Silph Road app working: basic, nginx, php-fpm. Basic is essentially a given, it's the scaffolding that makes charm layers work. NGINX and php-fpm are the two other components that need to be installed on each Silph Road app machine in order for the site to work properly; we'll glue these components together in the next section.

The next two are the interface declarations. These declarations make sure the libraries for these connections get compiled into the charm. They facilitate the implementation of the communication layer between charms. In this case, by declaring the MySQL interface we'll be able to get MySQL credentials from a charm which provides the MySQL interface. This allows us to automate deployments and scale without having to hand craft configuration files for each machine.

Finally, the reactive file! This is the heart of the automation of the deployment. In it, describes the instruction set on not only how to install the application, but how to maintain the application over time. While I'm using Python here, other languages are supported. I'll show the end result of the file and break down each chunk:

{% highlight python %}
import os
import tarfile

from charmhelpers.core.hookenv import (
    config,
    status_set,
    open_port,
)

from charms.reactive import (
    when,
    when_not,
    set_flag,
    remove_flag,
)

from charmhelpers.core.host import chownr
from charmhelpers.core.templating import render

from charms.layer import (
    php,
    git,
    nginx,
)


@when('php.ready')
@when_not('silph-road.installed')
def install_thesilphroad():
    web_path = '/srv/web'

    git.clone(config().get('source'), web_path)

    create_paths = {
        'app/tmp/cache/pokedex_cache': 0o777,
        'app/tmp/cache/persistent': 0o777,
        'app/tmp/cache/models': 0o777,
        'app/tmp/logs': 0o777,
        'app/webroot/img/user-verification-photos': 0o777,
    }

    for path, perm in create_paths.items():
        os.makedirs(os.path.join(web_path, path), perm)

    chownr(web_path, 'www-data', 'www-data')
    set_flag('silph-road.installed')


@when('config.changed.source')
def update_cfg():
    remove_flag('silph-road.installed')
    remove_flag('silph-road.ready')


@when('nginx.available')
@when('silph-road.installed')
@when_not('silph-road.nginx.ready')
def setup_vhost():
    nginx.configure_site(
        'silphroad',
        'silphroad-vhost.conf',
        socket=php.socket(),
    )

    service_restart('nginx')
    set_flag('silph-road.nginx.ready')


@when('silph-road.installed')
@when('database.available')
@when_not('silph-road.db.ready')
def configure_db(db):
    ctx = {'database': db}

    render(source='database.php.j2',
           target='/srv/web/app/Config/database.php',
           owner='www-data',
           group='www-data',
           perms=0o640,
           context=ctx
    )

    set_flag('silph-road.db.ready')


@when('silph-road.nginx.ready')
@when('silph-road.db.ready')
@when_not('silph-road.ready')
def ready_freddie():
    open_port(80)
    status_set('active', 'ready')
    set_flag('silph-road.ready')


@when('silph-road.ready')
@when('website.available')
def send_port(http):
    http.configure(80)
{% endhighlight %}

Here's that 100 lines of Python code. To get a few logistics out of the way, and of those not familiar with Python. The reactive system provides a means of when you can set and remove flags then declare what code to execute whenever a set of flags match. To declare which set of code gets executed when, we use [Python decorators]() which are all the lines prefixed with `@`. In the reactive framework `@when` and `@when_not` are equivalent to true or false. Multiple decorators stacked over a method are evaluated with the AND operator. So `@when('hello')` and a `@when_not('world')` means whenever the "hello" flag is set but the "world" flag isn't then the method it's decorating will be executed.

Remember when we included `layer:php-fpm` and `layer:nginx`? If you notice a few flags aren't explicitly set in the above python file yet we're responding to them. This is because each layer included has it's own reactive python file which is compiled into the charm at build time. Each of these reactive files have their own set of declarations that are globally accessible. This means each layer will run through it's set of steps to install the bits it needs. All we need to do is react to the `php.ready` and `nginx.available` states which are set from each layer respectively.

{% highlight python %}
@when('php.ready')
@when_not('silph-road.installed')
def install_thesilphroad():
    web_path = '/srv/web'

    git.clone(config().get('source'), web_path)

    create_paths = {
        'app/tmp/cache/pokedex_cache': 0o777,
        'app/tmp/cache/persistent': 0o777,
        'app/tmp/cache/models': 0o777,
        'app/tmp/logs': 0o777,
        'app/webroot/img/user-verification-photos': 0o777,
    }

    for path, perm in create_paths.items():
        os.makedirs(os.path.join(web_path, path), perm)

    chownr(web_path, 'www-data', 'www-data')
    set_flag('silph-road.installed')
{% endhighlight %}

The above chunk of code is the first declaration in the file. It is also the one that does most the heavy lifting for the charm. It's triggered whenever the `php.ready` flag is set and `silph-road.installed` flag isn't set. First we clone the source code repo which is declared as a configuration on the charm. Next, we create the additional file paths which aren't in the source code repo but are needed in order for the app to run. Finally we make sure the source code is owned by the right user before setting the `silph-road.installed` state.

{% highlight python %}
@when('config.changed.source')
def update_cfg():
    remove_flag('silph-road.installed')
    remove_flag('silph-road.ready')
{% endhighlight %}

This is a pretty simple hack, whever the source config key is changed, we remove the two main flags so that it runs through the installation again. Since each method is idempotent and the `git.clone()` method will run a `git pull` if the source directory exists it's a super easy way to get changes in the repo propagated. In the future I expect we'll grow this out to handle migrations in dataschema, for now there's not a need for this.

{% highlight python %}
@when('nginx.available')
@when('silph-road.installed')
@when_not('silph-road.nginx.ready')
def setup_vhost():
    nginx.configure_site(
        'silphroad',
        'silphroad-vhost.conf',
        socket=php.socket(),
    )

    service_restart('nginx')
    set_flag('silph-road.nginx.ready')
{% endhighlight %}

The next bit here glues up the NGINX configuration file for our specific site. Not shown in the tree above is a templates direcotry which has a `silphroad-vhost.conf`. The virtualhost configuration file isn't really all that interesting, it just sets up where the root directory is and glues the PHP-FPM socker/address to the reverse proxy. After rendering the configuration, we'll restart NGINX and set a new flag to avoid creating an NGINX configuration file over and over again.

{% highlight python %}
@when('silph-road.installed')
@when('database.available')
@when_not('silph-road.db.ready')
def configure_db(db):
    ctx = {'database': db}

    render(source='database.php.j2',
           target='/srv/web/app/Config/database.php',
           owner='www-data',
           group='www-data',
           perms=0o640,
           context=ctx
    )

    set_flag('silph-road.db.ready')
{% endhighlight %}

This block is similar to the NGINX one above, but is unique because it executes whenever the database relation is created. This flag exists because we included both the `database` relationship with the `mysql` interface and the mysql interface layer. These factors combined make it so when you deploy a MySQL compatible charm and create a relation between it and another charm, the database charm will create a set of credentials and transmit them directly to our charm. The mysql interface layer collects, validates, then raises the `database.available` flag. Since interface layers are special, they pass a context to the method we execute with a reference to the data from that relationship. `db` will actually have a `hostname`, `port`, `user`, `password`, and `database` keys set. The render method is a shortcut to a Jinja2 templating library and allow us to have a file in templates named `database.php.j2` which looks like this:

{% highlight php %}
{% raw %}
<?php

class DATABASE_CONFIG {
        public $default = array(
                'datasource' => 'Database/Mysql',
                'persistent' => false,
                'host' => '{{ database.host() }}',
                'login' => '{{ database.user() }}',
                'password' => '{{ database.password() }}',
                'database' => '{{ database.database() }}',
                'prefix' => '',
                'encoding' => 'utf8',
        );
}
{% endraw %}
{% endhighlight %}

Now we can dynamically create our database configuration without having to manually munge any files! Finally, we have the next two methods.

{% highlight python %}
@when('silph-road.nginx.ready')
@when('silph-road.db.ready')
@when_not('silph-road.ready')
def ready_freddie():
    open_port(80)
    status_set('active', 'ready')
    set_flag('silph-road.ready')


@when('silph-road.ready')
@when('website.available')
def send_port(http):
    http.configure(80)
{% endhighlight %}

The first one works to just aggregate all the previous states to make sure dependencies are satisfied. In this case we need to make sure nginx and the database are ready. Once that's done, we tell Juju to open port 80, set the status of the charm to active, and create the `silph-road.ready` flag. The second method glues the last layer and relationship we creatd. If you recall we have a relationship called `website` which implements the `http` interface and we included `interface:http` - this is the library that implements the communicaiton protocol. Unlike the `mysql` interface, which provides us a mysql endpoint we're providing an http endpoint. As such, we need to configure the relationship so that applicaiton consuming our http endpoint get the proper configuration. For the HTTP interface, this is as simple as providing the portname for our http server (the interface layer will provide the proper hostname, etc). So when `silph-road.ready` and `website.available` we'll take the context passed as a parameter and relay our port through it.

So why go through all this trouble? Afterall we're fighting aginst an already over-provisioned speed is of the essence. One of the biggest time sucks I've found is when you create bespoke, snowflake, infrastructure. It's fragile, brittle, and typically leads to problems down the road. By making things repeatable and leveraging constructs like Juju and Charms it makes our infrastructure not only repeatable, but also portable. Deploying The Silph Road takes just a few minutes and looks something like this:

{% highlight bash %}
charm build ~/silphco/silph-road-charm   # compile our layer to produce a charm
juju bootstrap silph.io-prod0            # create a juju controller
juju deploy mariadb                      # deploy our MySQL compatible service
juju deploy haproxy                      # our loadbalancer
juju deploy ~/silphco/builds/silph-road  # the compiled charm output
juju add-relation haproxy silph-road     # connect haproxy to silphroad
juju add-relation mariadb silph-road     # connect mariadb to silphroad
juju add-unit -n3 silph-road             # start scaling out!
{% endhighlight %}

I don't expect this will be the final version of this charm. It will certiainly take some tweaking as we venture further into the deployment. Making patches, however, is as easy as edit layer, charm build, juju upgrade-charm!
