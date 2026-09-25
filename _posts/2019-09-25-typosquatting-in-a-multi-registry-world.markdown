---
layout: post
title: "Container Image Squatting in a Multi-Registry World"
date: 2019-09-25 18:10:39 +0100
comments: false
categories: Docker
---

I've been starting to have a look at [podman](https://podman.io/) recently and in doing so, I noticed something potentially interesting from a security perspective, which is how podman handles the pulling of new container images.  As podman is billed as a "drop-in" replacement for Docker (and indeed provides a package to alias docker commands to their podman equivalents), it's interesting to note how default settings might differ, as these differences could trip up unsuspecting users moving from Docker to podman.

One of the more fixed things in the Docker landscape is how it address container image registries. Docker hub is hard coded in the source code of Docker as the default registry, so that if you do something like `docker pull nginx` or `docker pull raesene/alpine-containertools` it will assume that you're looking for these images on Docker Hub.  If you want to pull from an alternate registry, you need to provide the hostname and port of that registry server.

This behaviour appears to have been a source of frustration for other organizations in the container ecosystem, so it's not really a surprise to find that it works differently when you use other tools, like podman.

What podman does is allow users to specify their registry search order via a configuration file stored at `/etc/containers/registries.conf`. On a CentOS8 install the default registry search order is this

```
[registries.search]
registries = ['registry.redhat.io', 'quay.io', 'docker.io']
```

From that, we can see that both Redhat's registry and Quay come in before Docker.

This has a potentially interesting side effect from a security perspective, which is, if a user requests an image without specifying the host name and port of the registry, they could get the wrong image.

To give an example. On Docker hub there's an image I have with some container tooling called [alpine-containertools](https://hub.docker.com/r/raesene/alpine-containertools).  As an experiment I tried creating the same image name with totally different content on Quay.io which is [here](https://quay.io/repository/raesene/alpine-containertools).

So if you use podman and run `docker pull raesene/alpine-containertools` you're going to get the wrong image.

This could lead to attackers essentially squatting on common Docker Hub accounts to try and trick users into pulling malicious images.  A process made somewhat easier by the fact that you can register organization names on Quay.io (for example, I registered [nccgroup](https://quay.io/organization/nccgroup)).

## What can you do about this?

Well if you're planning to adopt podman there's a couple of things you could do to mitigate this risk

- Change the search order in registries.conf to have `docker.io` first, which essentially restores the default behaviour (although it could still produce unexpected results if you misspell your pull request)
- Remove the other registries from the file altogether (which essentially makes the behaviour the same as Docker)
- Ensure that you've registered any names you use on Docker Hub on Quay.io (this only works where you're pulling Docker Hub images from accounts you control, but of course who pulls Docker Hub images from accounts they don't control!)
- Use FQDNs for all your `pull` or `run` statements, essentially bypassing the search order.
- Make use of image signing to check that the image pulled is the same as that which was expected.