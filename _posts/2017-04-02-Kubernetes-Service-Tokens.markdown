---
layout: post
title: "Kubernetes Attack Surface - Service Tokens"
date: 2017-04-02 18:05:39 +0100
comments: false
categories: Kubernetes
---

Whilst spending some more time looking at Kubernetes, to help out with the forthcoming CIS Security standard, I was looking at cluster component authentication and noticed something that might not be known by everyone using Kubernetes, so I thought it'd be worth a post.

When pods are deployed to a cluster, in most default installs, the [Admission Contoller](https://kubernetes.io/docs/admin/admission-controllers/) will run and take a set of pre-defined actions before the pods go live.  One of those actions is to mount a [Service Account](https://kubernetes.io/docs/admin/service-accounts-admin/) inside the containers that make up the pod.

This service account includes a token which is mounted at a predictable location `/var/run/secrets/kubernetes.io/serviceaccount/token` .

What's interesting is that, by default unless [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) is deployed, it's likely that this token provides cluster admin privileges.

This means that any attacker with access to a container can, fairly easily, get full access to the cluster API (in fact it's kind of easier than the [kubelet exploit](https://raesene.github.io/blog/2016/10/08/Kubernetes-From-Container-To-Cluster/) ).

If you want to check this to see if it affects your cluster, just run a pod inside the cluster, attach to one of the containers, get a copy of [kubectl](https://storage.googleapis.com/kubernetes-release/release/v1.6.0/bin/linux/amd64/kubectl) and point it at your API Server with something like

`./kubectl config set-cluster test --server=https://[API_SERVER_IP]:[API_SERVER_PORT]`

then try out some kubernetes commands...

Fortunately this issue has been addressed with Kubernetes 1.6 setups which make use of the default RBAC policy, so if you're concerned about container breakout scenarios, I'd thoroughly recommend upgrading and making sure that you're using a restrictive RBAC policy.

