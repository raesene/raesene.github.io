---
layout: post
title: "Variance of defaults - Microk8s RBAC"
date: 2026-03-11 10:00:00 +0100
comments: false
categories: Kubernetes
---

One of the points I tend to make in my talks about Kubernetes security is that it's quite difficult to talk about what the security defaults are, as there are over 150 different Kubernetes distributions and services and each one of them has a different idea of what their security defaults should be.

I was recently dealing with a really good example of this so I thought it was worth writing up in a little detail, especially as I had mentioned it to a couple of other people in the Kubernetes space who were surprised to hear about it.

## Microk8s RBAC

[Microk8s](https://canonical.com/microk8s) is a Kubernetes distribution from Canonical, which is intended for use in a number of production scenarios. The unusual security default in this case is that, out of the box, it does not enable RBAC in your Kubernetes cluster! If you want to have RBAC enabled you have to enable it as an add-on after installation.

There are two knock-on effects here. The first is that every user defined with legitimate credentials is automatically given effective `cluster-admin` rights to the cluster. 

The second is that, by default, every workload deployed to the cluster will get a service account token, and every one of those service account tokens will be `cluster-admin`. So anyone who has access to, or can compromise, a single workload in the cluster will automatically get to be an admin in the cluster!

One of the trickier aspects of this is that Kubernetes will happily accept (cluster)role and (cluster)rolebinding manifests and create the objects, but those objects just won't have any effect.

Of course the correct resolution of this is to ensure that the first action you take after installing Microk8s is to enable RBAC.

## Canonical's response

To me, this is not a good default for Kubernetes security, so before writing it up, I decided to report it to Canonical, with details on why I felt this was not a good default. They were very friendly, but ultimately responded that this was a known and accepted default value. Their exact response was

```
Microk8s has RBAC disabled by default due to legacy compatibility reasons. RBAC was not initially available when Microk8s was first released, and therefore there are no plans to change this behavior.

There has been a GitHub issue opened regarding this, which you can find here: https://github.com/canonical/microk8s/issues/5400

If RBAC is required, users can run `microk8s enable rbac` after installation. However, the decision to do so ultimately falls on the user, which depends on that particular user's needs.
```

From my perspective this felt a little surprising as Kubernetes has had RBAC enabled by default since (IIRC) 1.8 in 2017. Whilst a commitment to backwards compatibility is admirable, there is a tradeoff here, which is that cluster operators expect to have RBAC working in their clusters, as it does with (AFAIK) every other Kubernetes distribution

## Conclusion

This case illustrates one of the big challenges in Kubernetes security, which is the sheer variety of software we have to deal with. It's important not to assume that just because a Kubernetes distribution is long-standing and widely used, that it will have hardened defaults.