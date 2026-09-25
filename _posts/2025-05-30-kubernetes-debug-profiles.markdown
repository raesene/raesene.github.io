---
layout: post
title: "Kubernetes Debug Profiles"
date: 2025-05-30 17:00:00 +0100
comments: false
categories: Kubernetes
---

I got a lesson today in the idea that it's always worth re-visiting things you've used in the past to see how they've changed, as sometimes there will be cool new features! 

In my [Kubernetes Post-Exploitation talk](https://youtu.be/4L8Dg_QSx30?si=hwH6LcwvXGCOVkhg) I make use of `kubectl debug` as a means to get a root shell on a cluster node. It's a very handy command but *I thought* it wasn't possible to use `ctr` commands from inside the shell you get with `kubectl debug` and that turns out to be outdated information!

## What's the problem?

If you've done much with container pentesting or offensive security, you'll have come across the idea that access to the Docker socket effectively gives root access to the underlying host via [The most pointless Docker command ever](https://zwischenzugs.com/2015/06/24/the-most-pointless-docker-command-ever/), and this is true even if you just have a container with that file mounted in.

However in modern Kubernetes clusters, it's likely that the underlying container runtime is [containerd](https://containerd.io/) and not Docker. What can be surprising is that the containerd socket works very differently than the Docker one. It assumes that the client program and the containerd server are operating on the same host with the same environment.

## (old) kubectl debug

This problem shows up when using the "legacy" profile for kubectl debug node (which is the default if you don't specify one).  Some commands, using the `ctr` client will work just fine, so things like pulling new images, however when you try to run a new container you'll get an error like this

```bash
ctr: failed to unmount /tmp/containerd-mount2094132404: operation not permitted: failed to mount /tmp/containerd-mount2094132404: operation not permitted
```

## Kubectl debug profiles to the rescue!

Fortunately Kubernetes SIG-CLI have been improving on the initial kubectl debug command by having a set of profiles that you can specify, which provide different sets of rights on the node you're debugging. The list of available profiles is "legacy", "general", "baseline", "netadmin", "restricted" or "sysadmin", with the default being "legacy".

So I decided to try the commands from my demo, but with the `sysadmin` profile specified as an option, and it works!

This is very handy if you're a sysadmin who wants to interact with the containerd socket as part of your troubleshooting, or if you're an attacker who's got access to a host and wants to hide some tools in a containerd container!

There are some details on what each of the profiles sets in terms of security options in this [KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-cli/1441-kubectl-debug#debugging-profiles)

## Conclusion

As ever there's loads of cool new Kubernetes features that come up all the time. I've been doing container security things for 9+ years now and I'm still finding interesting things to look at!