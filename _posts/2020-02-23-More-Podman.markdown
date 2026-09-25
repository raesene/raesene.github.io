---
layout: post
title: "More Podman - Rootfull containers, Networking and processes" 
date: 2020-02-23 12:10:39 +0100
comments: false
categories: Podman
---

# Introduction

This is a follow on from my [previous post](https://raesene.github.io/blog/2020/02/01/Comparing-Docker-And-Podman/) which started looking at how podman varies from running local containers with Docker.

One point that was raised after that post, was that podman can run containers as root as well, and that's an interesting area to explore.

## Running podman as root

So we can use `sudo` on an ubuntu host to run podman containers as the root user. There's a couple of reasons you might want to do this.  First up would be that you need access to functions that are not available to your standard user (for example binding ports < 1024), another could be down to the differences in network behaviour between rootless and rootfull containers.

One thing you'll notice immediately when using sudo to run podman containers is that, you're not sharing any container images with your ordinary user, so it'll pull down images that aren't present for the root user.

Unlike Docker there's no shared system level image repository, by default, so it will go and retrieve images where needed.

## Difference in networking - rootless v.s. rootfull

Another area where there are some notable differences between rootless and rootfull containers under podman is in networking.  As mentioned last time rootless containers use `slirp4netns` to provide containers an IP address.  This is quite a different model to the Docker bridge, with a couple of practical effects.

- There's no shared network for rootless containers.  Each one is plumbed into a `tap` interface which is then networked out to the host by slirp4netns.  So if you start two containers in rootless mode, by default, they can't talk directly to each other without exposing ports on the host.
- All your containers get the same IP address.  On the installation I'm using they all get `10.0.2.100` 

By comparison, if you run containers rootfully, the networking looks much more similar to the default Docker configuration. Containers will get an individual IP address, and will be able to communicate with other containers on the bridge network that they've been connected to.  

## Podman Networking - Usable MacVLAN

Anyone who's dug around in Docker networking for a while, will likely have come across the MacVLAN network type.  This is kind of like a VMWare Workstation/Fusion bridge network, where the container gets placed on the the same network as the host, getting rid of the need for explicit port forwarding.

The downside to Docker's implementation has always been that there's no support for DHCP with MacVLAN, so you need to have a range of IP addresses to assign to the container that aren't going to be used elsewhere.

When I was looking round Podman's networking options, I noticed that there's a plug-in that can allow Rootfull podman containers to do something similar, but with DHCP support.  This basically follows [this post](https://www.redhat.com/sysadmin/leasing-ips-podman) on the Redhat blog, but there's a couple of details that worked differently on the system I was running on.

After creating a network config, as mentioned in the post you start the DHCP plug-in.  On ubuntu this is in `/opt/cni/bin` instead of `/usr/libexec/cni`

You can then run a container attached to that network and it'll get an IP address on the same LAN as your host.  In my set-up this took a couple of seconds to do and you get some "network is down" messages as it's starting, which can be a bit disconcerting, but it does work!

## Rootless containers - and user sessions

One thing I noted whilst using Podman this week, which was a surprise to me was that containers launched by a user survive that user logging out and back in again. I had assumed that the container would come under the user session that launched them, and so would be terminated when the user logged out, however that's not what happens, the container will keep running and be available when you log back in, which is handy.

