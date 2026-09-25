---
layout: post
title: "Docker 1.12 - Macvlan"
date: 2016-07-23 10:05:39 +0100
comments: false
categories: Docker
---

Another new cool facet of the 1.12 release of Docker Engine is that Macvlan and Ipvlan support is leaving experimental and is available for all users.  So now instead of the rather convoluted procedure I mentioned [last time I looked at this](https://raesene.github.io/blog/2016/02/07/Exploration-in-Docker-bridging/) we can now simplify the setup of containers attached to the same network as the host, removing the need for NAT translation from the container network to the host network.  

An aside first is that it can get a little confusing with Docker's naming as you may be thinking "hey I already have containers on the docker bridge isn't that already like being connected to the host network?".  Well as explained [here](http://hicu.be/docker-container-network-types) what docker calls "bridged" is really "NAT" and macvlan is what most people would think of as "bridged"!

Anyway, the first step in getting a container online directly with the host interface is to create a new macvlan network

```
docker network create -d macvlan --subnet=192.168.41.0/24 --gateway=192.168.41.1 --ip-range=192.168.41.128/26 -o parent=ens33 testnet
```

So in this example our host network's subnet is 192.168.41.0/24 and our gateway is just the default gateway for that network.  Where it gets a little interesting is that we need to specify an ip range that docker can use to hand out to containers that connect to this network.  At the moment there's no supported way to get containers connecting to a macvlan network to use the host network's DHCP server so you need to either specify an `--ip-range ` with a range that's available and doesn't overlap with a DHCP scope on the host network or alternatively you can pass `--ip <address>` to the individual `docker run` commands that we use to create containers on this network.

Once you've created the network it's just a question of specifying the network to use in the docker run command, so something like

```
docker run -it --network=testnet ubuntu:16.04 /bin/bash
```

should give you a container, likely with an address of 192.168.41.128 in this case.

A couple of other points worth noting about macvlan support.  You can only have one macvlan or ipvlan network setup on a Docker engine instance at the moment (although obviously you can have multiple containers connected to it), and from the macvlan network, you won't be able to contact the IP address of the host system.

For more reading on this topic there's a great set of articles at hicu.be on [macvlan networking](http://hicu.be/docker-networking-macvlan-bridge-mode-configuration) and [bridges vs macvlan](http://hicu.be/bridge-vs-macvlan) amongst others.

