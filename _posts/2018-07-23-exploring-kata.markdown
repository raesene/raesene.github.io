---
layout: post
title: "Exploring Kata Containers"
date: 2018-07-23 18:45:39 +0100
comments: false
categories: Containers
---

This is the second part of a series, taking a brief look at some alternate container runtimes, which can be used with Docker and Kubernetes, the first part is [here](https://raesene.github.io/blog/2018/07/22/exploring-gvisor/).

## Introduction

[Kata Containers](https://katacontainers.io/) is a project to provide a container runtime which makes use of qemu virtualization to provide isolation for the contained processes.  At face value this seems a bit of an odd decision as generally companies have moved from virtualization based isolation to process based isolation with projects like Docker.

However having the flexibility to run some workloads with additional isolation is a useful option, and it's perfectly possible to have a single Docker engine instance which supports multiple container runtimes.

## Installation Notes

The Kata container installation process is pretty straightfoward. I was installing on Ubuntu so followed the instructions [here](https://github.com/kata-containers/documentation/blob/master/install/ubuntu-installation-guide.md).  One first note is that 18.04 is supported even though the docs currently say 16.04 or 17.10.

Once you've got the packages installed you need to configure the Docker daemon to use the new runtime.  Kata Containers provide some documentation on that [here](https://github.com/kata-containers/documentation/blob/master/install/docker/ubuntu-docker-install.md) however I went a slightly different route.

Their install process modifies the systemd unit file to add the runtime there and make it the default, but as I'm running a host with multiple container runtimes, it seemed like a better idea to make the change in Docker's daemon.json file which lives in /etc/docker/ .  I've got gVisor setup on this host as well so my file looks like this

```
{
    "runtimes": {
        "runsc": {
            "path": "/usr/local/bin/runsc"
        },
        "kata-runtime": {
            "path": "/usr/bin/kata-runtime"
        }
    }
}
```

One other important installation note is that, if you're setting up inside a VM, you'll need to enable nested virtualization, so that qemu will start ok.

## Usage

Once you've got it installed running a container with Kata Containers is as simple as adding `--runtime=kata-runtime` to the docker run command.  I think part of the allure of using something like Kata Containers is that you can still take advantage of the containerization workflows, without potentially reducing the security level that you've traditionally had with a VM based model.

## Notes

Once you've got your Kata Containers container up and running, there's a couple of things to notice.  The kernel version inside the container is likely to be different from that outside, which is kind of expected given that we're running in a VM as opposed to using Linux isolation.

**output of uname -a without kata containers**
```
Linux 41665d9e7de6 4.15.0-23-generic #25-Ubuntu SMP Wed May 23 18:02:16 UTC 2018 x86_64 Linux
```

**output of uname -a with kata containers**
```
Linux 1941e8a8e06a 4.14.51-132.container #1 SMP Tue Jul 3 17:13:46 UTC 2018 x86_64 Linux
```

Interestingly the text **container** in the kernel version could be a useful fingerprinting indicator.

As with gVisor there's a difference in the contents of `/proc` as well.  In a standard container I'm seeing 4700 entries against 2741 in the kata-containers version, so there's likely some exploration there to see what's different.

Getting information about what Kata containers is up to seems easy enough. There's a handy kata-env command that can be run `/usr/bin/kata-runtime kata-env` which outputs a load of useful information including things like what VM image is being used by qemu for the containers you are running.

Each container you run up spawns a kata-shim, kata-proxy and qemu process, there's details on exactly what each does in the project's [architecture docs](https://github.com/kata-containers/documentation/blob/master/architecture.md)

`--privileged` doesn't work under kata containers as with qemu isolation it doesn't make a great deal of sense to have a privileged mode. Also `--net=host` doesn't work and indeed it'll hang the hosts network quite effectively if you try! `--pid=host` doesn't work either, but at least it doesn't crash the host :) There's a document tracking limitations [here](https://github.com/kata-containers/documentation/blob/master/Limitations.md)

## Performance

I think it's fair to say that there's a bit of performance hit to using Kata Containers over standard Docker.  running an alpine:3.7 container using Docker shows an output from `sudo pmap -x [pid]` of 1.5MB . Running the same for kata containers and you get 3GB for the Qemu process and 600MB for the kata-shim process, so similar to what you'd see for VMs which is somewhat unsurprising.

Whilst I'm sure that there's going to be circumstances where that tradeoff will be worth it, that's a pretty significant impact if you're moving to containerization for the performance benefits.