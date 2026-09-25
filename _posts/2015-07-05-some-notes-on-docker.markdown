---
layout: post
title: "Some notes on docker"
date: 2015-07-05 20:10:15 +0100
comments: false
categories: Docker
---

I've been spending some time this weekend looking more at docker and where I think it could be useful for my workflows, and along the way I've learned a couple of things which I didn't know, so I thought it would be worth recording them, in case they're useful to others.  None of this is particularly earth shattering but hey could save someone some time :)

## Images and Containers

First was the distinction between images and containers.  Images are what you build from (e.g. a template) containers are running instances of those images.

## Container Persistence

If you do one of the canonical examples that's provided in many tutorials

```
docker run -t -i ubuntu:latest /bin/bash
```

You get an interactive shell prompt in a docker container.  If you exit that process you go back to the host and if you repeat the command you get another instance of the container, with none of the files you created in the first one.

From that you might get the impression that docker container filesystems are not persistent (i.e. when the exit all the data is lost), but that's not the case from what I can see. 

To get back into a container instance you've used previously you need to get the name or ID first.  To do that use

```
docker ps -a
```

which will list all the container instances you've ever run on that system and it'll look something like

```
CONTAINER ID        IMAGE                        COMMAND                CREATED             STATUS                      PORTS                    NAMES
39c1e5cf8319        raesene/auto-docker-dradis   "bundle exec rails s   47 minutes ago      Up 47 minutes               0.0.0.0:3000->3000/tcp   cranky_kilby
e529903a3183        raesene/auto-docker-dradis   "bundle exec rails s   55 minutes ago                                                           compassionate_yonath
f084aaca32ee        raesene/dradisframework      "bundle exec rails s   24 hours ago        Exited (0) 47 minutes ago                            agitated_ptolemy
8a8f7b95082a        2bfb337a0aa8                 "bundle exec rails s   24 hours ago        Exited (0) 24 hours ago                              fervent_hopper
8a2a0046b769        ruby:latest                  "/bin/bash"            24 hours ago        Exited (0) 24 hours ago                         	 compassionate_goodall
```

from that you can see each container has an ID (the first column) and a name (the last column).  You can restart the container using either of those.  For example to restart the container with the name compassionate_goodall to get some files off it, you'd use

```
docker start compassionate_goodall
docker attach compassionate_goodall
```

## docker hub build options

There's two options for getting your builds onto docker hub.  The first is to get a container, make modifications to it on your own docker engine and then commit the changes back to your own repository on docker hub.  This works fine but is rather lacking in transparancy as to what went into the build and also doesn't really lend to others participating.

The other option is using Docker Hub automated builds, where a github/bitbucket repository is created with the Dockerfile present and then Docker Hub handles the build in an automated fashion from that Dockerfile. [docker automated build docs](https://docs.docker.com/docker-hub/builds/)


## Exposing docker ports

By default containers have no ports exposed the the host network and many containers do not have things like SSHD installed, so remote access isn't there by default.  The best way to map a port from the host to the guest is when the docker run command is used with the `-p` option.

