---
layout: post
title: "When is admin not admin?, when it's super-admin!" 
date: 2024-01-06 10:27:00 +0100
comments: false
categories: Kubernetes
---

I came across an interesting change in how Kubeadm based clusters handle initial credential setup in Kubernetes 1.29 and later, so thought it was worth a quick post. [Smarticu5](https://twitter.com/smarticu5) had a really unusual error, which was that on a newly created Kubeadm cluster he was getting a `forbidden` error when using the default `admin.conf` credential created by Kubeadm.

This specific error *shouldn't* be possible, as traditionally `admin.conf` is a member of the `system:masters` group which bypasses RBAC checks and always gets full cluster-admin level access to everything, so if a valid credential is presented, it should always work.

Interest piqued, we took a bit of a closer look at what was happening. First up was to decode the client certificate on the `admin.conf` file to see what usernames and groups it was associated with (If you're ever looking to do this and can't remember the exact commands, I'd recommend using [Cyber Chef](https://gchq.github.io/CyberChef/) which can base64 decode, and decode X.509 certs).

The output we got back was this:

```bash
  CN = kubernetes
Subject
  O  = kubeadm:cluster-admins
  CN = kubernetes-admin
```

Sooo this credential was no longer a member of `system:masters`, interesting! Next step, as is often the case when spelunking around in Kubernetes was to go look at Github to see if any recent changes had been made to how things work. Sure enough, there was a [PR](https://github.com/kubernetes/kubernetes/pull/121305) which explained how Kubeadm now has two files with initial credentials, the original `admin.conf` and a new `super-admin.conf`. The `admin.conf` has been changed to use an RBAC group for access, which should give it `cluster-admin` rights, but in a way that they could be revoked by removing the rights of that group, and then have the `super-admin.conf` file still be a member of `system:masters` whose rights can't be revoked by modifying RBAC.

Digging back from this PR, to the referenced issue to get some back-story, I got a slight surprise to find it was [one I'd opened](https://github.com/kubernetes/kubeadm/issues/2414) back in 2021 about not using system:masters!

## What does this mean?

There's a couple of practical consequences to consider with this change.

- If you're tracking permissions on sensitive files, and access to them, where you're using a `kubeadm` based Kubernetes distribution, you will need to update your tracking to include the new `super-admin.conf` file.
- The `super-admin.conf` file should *not* be used for any administrative tasks, instead place it somewhere safe and only use it if RBAC gets completely messed up. `admin.conf` isn't ideal either as it's a generic account, but at least its permissions can be revoked now!
- It is now possible to see a `forbidden` error when using `admin.conf` if RBAC isn't fully configured, or if the `clusterrolebinding` for the `kubeadm:cluster-admins` group has been changed.

## Conclusion

Another example of why it's important to keep up to date with changes in Kubernetes, and to understand how your cluster is configured. This change is a good one, as it makes it easier to revoke the rights of the initial credential, but it's important to understand how it works, and how it might impact your cluster.
