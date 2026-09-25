---
layout: post
title: "Escalating Away" 
date: 2020-12-12 14:00:39 +0100
comments: false
categories: Kubernetes
---

Following a recent run of the container security training course I do, I was poking around a bit with the escalate verb in Kubernetes RBAC and found some interesting points, so thought it'd be worth documenting, as it's not necessarily the best known part of RBAC.

The reason we were interested in the escalate verb is to answer the question "If I can get secrets in a standard kubeadm cluster, what are my options for escalating my privileges to cluster-admin?". This question used to be pretty straightforward as until last year there was a service account in the kube-system namespace which had cluster-admin rights, so you could easily get its secret and be cluster-admin. The service account in question is `clusterrole-aggregation-controller`. However last year the rights on this service account got changed (at least in part as I pointed it out). From [the commit](https://github.com/kubernetes/kubernetes/commit/8b155e82d876c8130962e61b2235f2bd066abde1) you can see that the cluster-admin rights were replaced with `escalate` rights on cluster roles.

So that leads to the question, "how can I use this permission?". Unlike the `impersonate` verb, there's no handy kubectl flags to add to instantly escalate your rights. So I was looking for information on the topic (which is pretty sparse) and found this [interesting blog post](https://www.impidio.com/blog/kubernetes-rbac-security-pitfalls), which led to the answer.

What escalate does is bypass the Kubernetes RBAC check which prevents users who are able to create roles or cluster roles from creating (or editing) these objects to have more rights than they do. So what we can do, once we have the service account token for the clusterrole-aggregation-controller is to edit our own cluster role to increase our permissions?

We can run `kubectl edit clusterrole system:controller:clusterrole-aggregation-controller` and then edit it to add this to the rules section 

```
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
```

and away you go!

## Conclusion

Things like `bind`, `escalate` and `impersonate` are lesser known features of RBAC but one that you should be aware of if you're a systems administrator or if you're looking at security tooling around Kubernetes.