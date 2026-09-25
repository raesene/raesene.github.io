---
layout: post
title: "Network Tools in Non-Root Docker Images"
date: 2017-07-23 13:15:39 +0100
comments: false
categories: Docker
---

As some environments which allow for Docker images to run (e.g. OpenShift Origin's default setup) don't allow containers to run as the root user, its worth knowing about other ways to get some networking and security tools run without having to have root.

Usually tools like nmap, tcpdump and ping will either need to be setuid root or be run as a user who has root level privileges, however with a bit of capabilities fiddling its relatively easy to get a container that doesn't need that level of privilege.

The key is use use the `setcap` utility to add the appropriate capability to your binaries, in this case CAP_NET_RAW.  CAP_NET_RAW is generally avaialable to containers as it's on Docker's default white list.

Once you've downloaded packages into the image, just use something like

```
RUN setcap cap_net_raw+ep /usr/bin/nmap
```

to set the capability on that binary.  One trick to note is that setcap doesn't work on symbolic links so you need to find the destination of any links before using it.  One example is that in Alpine based images, `/bin/ping` is just a symlink to `/bin/busybox` so if you want to enable CAP_NET_RAW in that setup `/bin/busybox` should be your target.

There's an example repo and associated Docker Hub image [here](https://github.com/raesene/alpine-noroot-containertools) and [here](https://hub.docker.com/r/raesene/alpine-noroot-containertools/)