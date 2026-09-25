---
layout: post
title: "Cap or no cap"
date: 2025-04-23 11:00:00 +0100
comments: false
categories: Kubernetes
---

I was looking at a [Kubernetes issue](https://github.com/kubernetes/kubernetes/issues/131336) the other day and it led me down a kind of interesting rabbit hole, so I thought it'd be worth sharing as I learned a couple of things.

## Background

The issue is to do with the interaction of `allowPrivilegeEscalation` and added capabilities in a Kubernetes workload specification. In the issue the reporter noted that if you add `CAP_SYS_ADMIN` to a manifest while setting `allowPrivilegeEscalation: false` it blocks the deploy but other capabilities when added do not block.

`allowPrivilegeEscalation` is kind of an interesting flag as it doesn't really do what the name says. In reality, what it does is set a specific Linux Kernel setting designed to stop a process from getting more privileges than when it started, however the name implies it's intended to do a more wide ranging set of blocks. My colleague Christophe has a [detailed post looking at this misunderstanding](https://blog.christophetd.fr/stop-worrying-about-allowprivilegeescalation/).

However what was specifically interesting to me was, when I tried out a quick manifest to re-create the problem, I wasn't able to and the pod I created was admitted ok.

After a bit of looking I realised that when adding the capability, I'd used the name `SYS_ADMIN` instead of `CAP_SYS_ADMIN`, and it had worked fine, weird!

## Exploring what's going on

I decided to put together a couple of quick test cases to understand what's happening (manifests are [here](https://github.com/raesene/k8sapecsa)). 

- `capsysadminpod.yaml` - This pod adds `CAP_SYS_ADMIN` to the capabilities list
- `sysadminpod.yaml` - This pod adds `SYS_ADMIN` to the capabilities list
- `dontallowprivesccapsysadminpod.yaml` - This has `allowPrivilegeEscalation: false` set and adds `CAP_SYS_ADMIN` to the capabilities list
- `dontallowprivescsysadminpod.yaml` - This has `allowPrivilegeEscalation: false` set and adds `SYS_ADMIN` to the capabilities list
- `invalidcap.yaml` - This pod has an invalid capability (`LOREM`) set.


Trying these manifests out in a [kind](https://kind.sigs.k8s.io/) cluster (using containerd as CRI) showed a couple of things

- Adding `CAP_SYS_ADMIN` worked but there was no capability added.
- Adding `SYS_ADMIN` worked and the capability was added.
- setting `allowPrivilegeEscalation: false` and adding `CAP_SYS_ADMIN` was blocked
- setting `allowPrivilegeEscalation: false` and adding `SYS_ADMIN` was allowed and the capability was added.
- setting an invalid capability worked ok but no capability was added.

So a couple of lessons from that. Kubernetes does not check what capabilities you add, and no error is generated if you add an invalid one, it just doesn't do anything. Also there's a redundant block in Kubernetes at the moment where something that doesn't do anything is blocked, but something which does do something is allowed ok...

Doing some more searching on Github turned up some more history on this. Back in 2021, there was a [PR to try and fix this](https://github.com/kubernetes/kubernetes/pull/105237) which didn't get merged, and there's another [issue from 2023](https://github.com/kubernetes/kubernetes/issues/119568) on it as well.

From that one thing that caught my eye was that apparently CRI-O handles this differently than containerd, which I thought was interesting

## Comparing CRI-O - with iximiuz labs

I wanted to test out this difference in behaviour, but unfortunately I don't have a CRI-O backed cluster available on my test lab. Fortunately, iximiuz labs has an awesome [Kubernetes playground](https://labs.iximiuz.com/playgrounds/k8s-omni) where you can specify various combinations of CRI and CNI to test out different scenarios, which is nice!

Testing out a cluster there with CRI-O confirmed that things are handled rather differently.

- Adding `CAP_SYS_ADMIN` worked and the capability was added.
- Adding `SYS_ADMIN` worked and the capability was added.
- setting `allowPrivilegeEscalation: false` and adding `CAP_SYS_ADMIN` was blocked
- setting `allowPrivilegeEscalation: false` and adding `SYS_ADMIN` was allowed and the capability was added.
- setting an invalid capability resulted in an error on container creation (CRI-O prepended the capability set with `CAP_` and then threw an error stopping pod creation as it was invalid).

So we can see that CRI-O handles things a bit differently, allowing both `SYS_ADMIN` and `CAP_SYS_ADMIN` to work and erroring out on invalid capabilities!

## Conclusion

Sometimes we can assume that Kubernetes clusters will work the same way, so we can freely move workloads from one to another, regardless of distribution. This case provides an illustration of one way that that assumption might not hold up, and we can see some surprising results!