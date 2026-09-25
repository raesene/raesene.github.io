---
layout: post
title: "Exploring gVisor"
date: 2018-07-22 18:45:39 +0100
comments: false
categories: Containers
---

As part of some talks I did for the recent NCC Con, I started looking at the [gVisor](https://github.com/google/gvisor) project from Google (nothing like having to write a presentation to provide motivation!).  

## Introduction

gVisor is an alternate Container runtime which replaces runc in the Docker stack with their runsc component.  It takes an interesting approach to container isolation which promises good performance and enhanced isolation over the base Docker experience.  Google have implemented a number of Linux syscalls in Go, so that the process running in the container doesn't directly need to communicate with the underlying Linux Kernel.  They have some more details on this on the [gVisor homepage](https://github.com/google/gvisor)

## Setup

The installation process is pretty straightforward, with a single binary to place on your Docker Engine host and then a quick modification to `/etc/docker/daemon.json` to make the alternate runtime available.

## Usage

After that you can start containers with the `--runtime=runsc` switch and they'll make use it.  Notably it's perfectly possible to run some containers on a host with standard Docker and others with gVisor at the same time.

## Notes

Once you get up and running with a gVisor container, there's a number of interesting things to note (well if you're interested in how container runtimes work anyway).

First up the kernel version. Generally with Linux containers, the kernel version inside a container is the same as outside as the container makes use of the same underlying kernel when it's operating.  However as gVisor is intercepting syscalls before they can get to the underlying Linux kernel, we see things slightly differently when it's in use.

If you run `uname -a` when running a gVisor container you get 

```
Linux 903108eb81ab 3.11.10 #1 SMP Fri Nov 29 10:47:50 PST 2013 x86_64 Linux
```

On the same host running without gVisor I get 

```
Linux 835730d3b41c 4.15.0-23-generic #25-Ubuntu SMP Wed May 23 18:02:16 UTC 2018 x86_64 Linux
```

which is the same as my underlying host (as expected).  This could be useful for fingerprinting the runtime in use as I'd expect that all containers running a given gVisor version will return the same info. from uname regardless of the underlying system.

Another place you'll see a difference is in `/proc` .  The gVisor project is working on exposing various pieces of informatio in proc (more information [here](https://github.com/google/gvisor/tree/master/pkg/sentry/fs/proc)) but at the moment there's a lot less info. here than you would find normally in a Docker container (which has positives and negatives).  As a quick metric `ls -laR` in /proc produces 304 entries in a gVisor container against 4700 in a standard Docker one.

One side note from this is that as a result of this, tools like [amicontained](https://github.com/genuinetools/amicontained) can't necessarily get the information they need to provide info about the security of the container, as they rely on reading information from files in /proc.

Another thing worth noting at this point is that gVisor is still relatively young as a project and does have bugs.  For example running `ls -laR | wc -l` reliably hung in an alpine:3.7 container when I was testing.

If you're looking to tell whether containers on a host are using gVisor or runc, the following command should return the relevant information

```
docker inspect --format='{% raw %}{{.HostConfig.Runtime}}{% endraw %}' [container_name]
```

## Performance

On the performance front, some very basic tests, show that there does appear to be some overhead in using gVisor, which may or may not be important to your usecase.

The test I did was to run 

```
docker run -it alpine:3.7 /bin/ash
```

with gVisor and runc. If you do that and run `docker stats` it shows the runc container using 1.16MiB of memory and the gVisor one using 84MiB ! Also there seems to be some constant CPU usage on the gVisor container, even though it's just running an idling ash shell with nothing else happening.




