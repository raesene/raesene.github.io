---
layout: post
title: "Auditing RBAC - Redux" 
date: 2022-08-14 09:10:39 +0100
comments: false
categories: Kubernetes
---

I was doing some reading on the topic of Kubernetes RBAC this week and I realised that a good article on the topic of auditing RBAC by [Mark Manning](https://twitter.com/antitree/) had unfortunately succumbed to bitrot (Although the wayback machine [still has a copy](https://web.archive.org/web/20200709085334/https://www.nccgroup.com/us/about-us/newsroom-and-events/blog/2019/august/tools-and-methods-for-auditing-kubernetes-rbac-policies/)), so I thought it would be a good opportunity to revisit the topic as there are some interesting nuances to it.

## The challenges of auditing Kubernetes authorization

The idea of auditing the rights that users and services have to the Kubernetes API is obviously an important one in terms of operating a secure cluster. Regular reviews of access to ensure that "least privilege" principles are a necessary part of securing your environment.

Kubernetes authorization is an additive process where the permissions from one or more "modes" are added together to see if a user can access a given resource. The most common of these is RBAC, however from the [documentation](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules) we can see that there are others (Node, ABAC and Webhook).

This means that in order to effectively audit a cluster's permissions you need to know which of the modes is enabled and be able to enumerate all the rights granted at each one. Typically (but [not always](https://github.com/Azure/AKS/issues/3004)) Node and RBAC will be enabled, but for some managed clusters (e.g. AKS and GKE) Webhook authorization will also be involved.

Auditing Node authorization is pretty simple, it's a specialist mode which has the purpose of restricting what kubelet credentials can do, which is a mechanism designed to reduce the risk of kubelet credentials being compromised.

Auditing Webhook authorization will depend on how the web service (for example Azure RBAC) being used works.

That leaves us with RBAC. This will pretty much always be enabled, so you'll need to audit it.

## RBAC Complexities

The first complexity to note is that RBAC works at two scopes, namespace and cluster. It uses `Role` and `RoleBinding` objects at the namespace level and `ClusterRole` and `ClusterRoleBinding` objects at the cluster level. There's three combinations of these objects possible, two of which are obvious and the third of which may be surprising.

- `Role` + `RoleBinding` == Rights provided at a namespace level
- `ClusterRole` + `ClusterRoleBinding` == Rights provided at a cluster level
- `ClusterRole` + `RoleBinding` == Rights provided at a namespace level

Within RBAC there are some other nuances to be aware of as well. Firstly Kubernetes does not have a fixed list of "verbs" that can be applied to resources, in addition to the expected CREATE, GET, PATCH, DELETE you can do things like [educate dolphins](https://blog.aquasec.com/kubernetes-verbs). Then in terms of the resources themselves, in addition to the standard set that are shipped as part of Kubernetes, there will likely be a large number of custom resources created by software installed in the cluster, for example most CNI providers will have several CRDs relating to networking objects.

So for a comprehensive audit, it's necessary to understand all the services which are plugged into the Kubernetes API (e.g. operators) and how they are set-up from an authorization perspective.

One other complication is that Kubernetes does not have user database so to find things like which users are in a group mentioned in RBAC, you'll need to audit the identity system(s) that are used by the cluster.

Given all that what tools do we have that can help us in our work?

## RBAC Tooling

### Kubectl

First up there are some in-built capabilities within `kubectl` which can be handy. The `kubectl auth can-i` command can be used as a starting point, although it's a bit limited when it comes to the unusual verbs we mentioned before.

Running `kubectl auth can-i --list` will provide a decent list of rights available via RBAC (but won't include rights through Webhook authorization)

```bash
kubectl auth can-i --list
Resources                                       Non-Resource URLs   Resource Names   Verbs
*.*                                             []                  []               [*]
                                                [*]                 []               [*]
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
                                                [/api/*]            []               [get]
                                                [/api]              []               [get]
                                                [/apis/*]           []               [get]
                                                [/apis]             []               [get]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/openapi/*]        []               [get]
                                                [/openapi]          []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
```

It's also possible (if you have the `impersonate` right) to run this command as other users to see what their rights look like. So for example to look up the rights of the `ttl-controller` in the `kube-system` namespace 

```bash
kubectl --as=system:serviceaccount:kube-system:ttl-controller auth can-i --list
Resources                                       Non-Resource URLs                     Resource Names   Verbs
events                                          []                                    []               [create patch update]
events.events.k8s.io                            []                                    []               [create patch update]
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
                                                [/.well-known/openid-configuration]   []               [get]
                                                [/api/*]                              []               [get]
                                                [/api]                                []               [get]
                                                [/apis/*]                             []               [get]
                                                [/apis]                               []               [get]
                                                [/healthz]                            []               [get]
                                                [/livez]                              []               [get]
                                                [/openapi/*]                          []               [get]
                                                [/openapi]                            []               [get]
                                                [/openid/v1/jwks]                     []               [get]
                                                [/readyz]                             []               [get]
                                                [/version/]                           []               [get]
                                                [/version]                            []               [get]
nodes                                           []                                    []               [list patch update watch]
```

### Third Party

In addition to using kubectl there are a wide variety of third party tools which can be used to help assess RBAC rights. There's a longer list [here](https://www.container-security.site/general_information/tools_list.html#rbac-assessment-tools) but a couple of interesting ones are :-

- [rbac-tool](https://github.com/alcideio/rbac-tool) - RBAC Toolbox from Alcide. This has a variety of very useful functions like graphing RBAC and doing an analysis for "risky" permissions granted to cluster users.
- [Kubiscan](https://github.com/cyberark/KubiScan) - Another tool which can scan for risky permissions, this one from Cyberark.
- [krane](https://github.com/appvia/krane) - RBAC static analysis and visualisation tool from Appvia.
- [RBAC Police](https://github.com/PaloAltoNetworks/rbac-police) - RBAC analysis tool from Unit42 with an extensible policy library.


## Conclusion

I think it's clear from this post that auditing Kubernetes authorization can be a bit of a tricky task. Kubernetes flexible architecture makes it difficult to provide generalized auditing tools as the way that clusters are implemented, with multiple authorization modes and the way the RBAC system works makes being definitive about user rights difficult.

Tools are a great way to get started with understanding cluster rights, but we always have to be aware that they can only tell you what they can see within the scope of their operation.