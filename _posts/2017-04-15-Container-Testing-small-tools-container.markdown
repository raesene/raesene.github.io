---
layout: post
title: "Container Testing - A small tools container with SSH"
date: 2017-04-15 13:35:39 +0100
comments: false
categories: Containers
---

When you're doing security testing of container environments one of the things that can be pretty useful is having a container with useful tools connected to the container network.  From there you can run network scans of the container network and also test the scenario of "malicious container"

There's a couple of ways of achieving this goal. You could run a container interactively with something like

`docker run -i -t [image_name] /bin/sh`

but one of the downsides of using a container like this is if you exit from the container, it'll stop running, so ideally you want a container that'll keep running and that you can connect to over the network.

One example of this approach is [this image](https://hub.docker.com/r/raesene/alpine-containertools/) which has a couple of useful features.

Its based on alpine linux so the image is nice and small, and it runs an SSH daemon so that it'll stay running and you can just connect in with SSH (assuming that you have a root from the location you start from to the container)

It also has a neat trick that I got from some [Fedora dockerfiles](https://github.com/fedora-cloud/Fedora-Dockerfiles/blob/master/ssh/entrypoint.sh) which is that it creates a random password each time you start an instance of the image.  This is a good thing (tm) as you don't want a static password baked in to your image.

To use this approach just do something like

`docker run -d raesene/alpine-containertools`

then find out what IP address has been assigned to your container with

`docker inspect -f "{% raw %}{{.NetworkSettings.IPAddress}}{% endraw %}" [container_name]`

and what the password that's been generated with 

`docker logs [container_name]`

and you should be able to SSH in and get started with testing.  This image has some network tools like nmap and curl and also some container tools like the docker client, kubectl and etcdctl which is handy for checking whether you can get access to the ETCD database in a Kubernetes network.