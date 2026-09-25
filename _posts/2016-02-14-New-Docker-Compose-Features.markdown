---
layout: post
title: "New Docker Compose Features"
date: 2016-02-14 17:15:39 +0100
comments: false
categories: Docker
---

Along with the new version of Docker Engine which came out recently there were some handy updates to Docker Compose.  Back when I started looking at using compose and Docker containers for [pen testing](https://raesene.github.io/blog/2015/08/15/set-up-a-complete-security-test-environment-with-command-and-docker-compose/) one of the drawbacks was that there was no great way to define a shared area for all the containers to save their data to as part of the compose setup.

So now with the new setup, we can use something like the example below, and then start it up with ```docker-compose run sectest /bin/bash```

This will start up four containers on the default private docker network, and provide a shell for the sectest container which we can use to run tools like nmap and metasploit.  Using this approach we get a couple of handy features.

Because of docker links the names of the other containers are registered in each one, so for example in sectest we can ```ping dradis``` and have that work ok.

As the browser and the dradis/openvas instances are on the same private network we can access these browser based tools without having to expose them to the wider network.

All the tools will have a directory in their filesystem at ```/data``` which points to our data volume.  At the end of the test this can then be saved off either from within the tools or outside at host level.

If you're doing it at host level you need to note where the data's actually stored.  To do that do

```docker volume ls```

to get the name of your data volume.  By default it'll be the name of the directory you ran the docker-compose file from and then '-data' after that.

Once you know the name just do ```docker inspect (name)``` and copy the files from there to whereever you're storing them.


{% highlight yaml %}
version: '2'

services:
  sectest: 
    image: raesene/sectest 
    command: /bin/bash 
    links: 
    - dradis 
    - openvas
    - firefox
    volumes:
      - data:/data
  dradis: 
    image: raesene/auto-docker-dradis 
    expose: 
      - "3000"
    command: bundle exec rails server 
    volumes:
      - data:/data
  openvas: 
    image: mikesplain/openvas 
    expose: 
      - "443"
      - "9390"
      - "9391"
    volumes:
      - data:/data
  firefox:
    image: jess/firefox
    links:
      - dradis
      - openvas
    environment:
      DISPLAY: $DISPLAY
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix
      - data:/data 
volumes:
  data: {}
{% endhighlight %}

