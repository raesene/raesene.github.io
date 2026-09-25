---
layout: post
title: "Containers and Low Ports" 
date: 2021-07-03 20:00:39 +0100
comments: false
categories: Containers
---

I had cause to look into unprivileged binding of ports in Docker containers recently, and found a couple of things that were a surprise to me, so taking the time to write them down, in case it's of use to anyone else.

## Background

Traditionally, on a Linux system, to bind a port lower than 1024 a process either had to be run by the root user, be setUID root or have CAP_NET_BIND_SERVICE. As a result of this, many containers images that want to use low ports would run as the root user. Additionally the Docker daemon included CAP_NET_BIND_SERVICE as one of the default capabilities provided to running containers.

## sysctls to the rescue!

Effectively the reasons that used to apply for having a set of "privileged" ports no longer really apply, so in version 4.11 of the Linux kernel a new option was introduced to allow system administrators to set the [`ip_unprivileged_port_start` sysctl](https://sysctl-explorer.net/net/ipv4/ip_unprivileged_port_start/) to a different value. By setting this to `0` you allow unprivileged processes to bind any port.

## State of play in container land

Looking into how this works in container-land, I got a pleasant surprise which is that the latest version already sets this by default (here's [the commit](https://github.com/moby/moby/pull/41030)), along with allowing ping sockets, so that ping works as an unprivileged user too. As this is a namespaced sysctl, it's only set in the net namespace of the container, and not set at the host level.

For Podman, the option isn't enabled by default, but can be set on a per-container basis with `--sysctl net.ipv4.ip_unprivileged_port_start=0`

For ContainerD there's an [open issue](https://github.com/containerd/containerd/issues/4936) to add the same default, and it's possible to add the setting using the same switch as for podman.

In the Kubernetes project there's also an [open issue](https://github.com/kubernetes/kubernetes/issues/102612) about enabling this by default.

## Conclusion

Allowing unprivileged users to bind low ports is likely to be a safer way to provide this functionality than using setUID root or using capabilities, so it's good to see things moving in that direction. At the moment I could see some confusion occuring due to differing defaults amongst containerization solutions, but hopefully in the longer run, it'll all get smoothed out :).