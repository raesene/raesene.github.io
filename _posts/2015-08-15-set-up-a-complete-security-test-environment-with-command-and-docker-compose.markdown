---
layout: post
title: "Set-up a Complete Security Test Environment with One Command and Docker Compose"
date: 2015-08-15 20:06:39 +0100
comments: false
categories: Docker
---

Following on from my last post on [Using Docker for Security Testing](http://raesene.github.io/blog/2015/07/23/using-docker-for-security-testing/), I thought it would be interesting to see if we can set-up an even more automated environment by using [Docker Compose](https://www.docker.com/docker-compose).  Docker Compose is a means of creating a linked set of containers, which you can configure to be started up together, so useful where you want to make use of multiple systems at the same time.

For the use case of Security Testing I was thinking it would be nice to have containers which provide a service that you connect to over a network port, for example [Dradis](http://dradisframework.org/) or [OpenVAS](http://www.openvas.org/), alongside your main command line driven container which has tools that you're more likely to use interactively on a test, for example [Nmap](https://nmap.org/) or [Metasploit](https://github.com/rapid7/metasploit-framework).

It turns out this is relatively straightforward using Docker Compose.  You can set-up a yml file with information on the containers you want to instantiate (the default name is docker-compose.yml) and then just spin that up with a single command.

So for this setup the Docker Compose file looks like this

{% highlight yaml %}
sectest:
  image: raesene/sectest
  command: /bin/bash
  links:
     - dradis
     - openvas
dradis: 
  image: raesene/auto-docker-dradis
  ports: 
      - "3000:3000"
  command: bundle exec rails server
openvas:
  image: mikesplain/openvas
  ports:
      - "443:443"
      - "9390:9390"
      - "9391:9391"
{% endhighlight %}

This file defines three containers that we're going to use.  The first one is an instance of a basic Security Testing container that I created [here](https://github.com/raesene/sectest).  There's nothing too major in there it just sets up some tools that I commonly use on reviews.  In that first stanza we specify that the dradis and openvas containers are linked to that one, which is relevant when we start things up.

In the next section we start up the dradis container and here it's worth noting two points.  The first is the "ports" command.  This sets up port mappings so that the service will be visible on the docker host, which is where we'll be connecting to it from.  The second thing to note is the "command" directive which just specifies what should be run when the container comes up.

In the third section we specify the openvas container and set-up some port mappings to expose the relevant ports to the host.

Now we've got this file created we just need to start things up.  The command I've used for this is

```
docker-compose run --service-ports sectest /bin/bash
```

What this does is start-up the sectest container and any linked containers (which is why it was important to add those links in the docker-compose.yml file), also the --service-ports option is specified so that the ports will be exposed, and lastly we specify the commmand to run, in this case /bin/bash.

If all goes well this should download and start-up all three containers and you should have a command prompt in the sectest container, along with started accessible services for the other two containers.

