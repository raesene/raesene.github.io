---
layout: post
title: "Fun with Kubernetes Authorization Auditing - multiple authz plugins" 
date: 2024-04-22 11:27:00 +0100
comments: false
categories: Kubernetes
---

One of the features of Kubernetes security, is its flexible model. This allows cluster operators to have multiple Authentication or Authorization modes running covering a number of use cases. This does introduce some complexity though both in terms of operation and also in terms of reviewing or auditing rights.

The most common case here is cloud managed Kubernetes where, in addition to the in-built RBAC authorization, you'll often find that there's a webhook authorization mode enabled as well to allow for integration with the cloud provider's IAM system.

## Multiple authorizer process

(Updated with information from [@pires](https://x.com/therealpires/status/1782729861951332712))

In principle the way that Kubernetes authorization works is that every configured authorizer is queried in series (with the order based on the order of the parameters provided to the `authorization-mode` flag on the API server) to see if a given action is explicitly allowed or explicitly denied. If an authorizer doesn't match with either an explicit allow or an explicit deny, the next configured authorizer is queried. So the effective rights will be based on the responses from one or more configured authorizers.

There's a couple of fun nuances of doing this, so I thought it was worth some discussion :)

## Auditing permissions in Kubernetes

I've covered some details of this [before](https://raesene.github.io/blog/2022/08/14/auditing-rbac-redux/), so we'll just look at the specific aspects of `kubectl auth can-i` as a mechanism for reviewing permissions. This command has two ways of being used the first allows for specific questions to be asked like `kubectl auth can-i get pods` and the second which lists all the permissions of the user `kubectl auth can-i --list` . So knowing this lets see what happens if we use this in a cluster with multiple authorization mechanisms setup.

## Setting up a webhook authorizer

For the purposes of this article I just wanted the most simplistic implementation of a webhook authorizer, so I put one together (with some help from an LLM). The code is [here](https://github.com/raesene/k8ssimpleauthzwebhook), it's basic but should help explain things.

## Auditing permissions with two authorizers

Once we have a cluster setup using this code, we can test things out. first I created a user called `jane` , using [teisteanas](https://github.com/raesene/teisteanas). With the user setup, I can use kubectl to get a list of all Jane's permissions

```
kubectl --kubeconfig=jane.kubeconfig auth can-i --list
Warning: the list may be incomplete: webhook authorizer does not support user rule resolution
Resources                                       Non-Resource URLs   Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                  []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
                                                [/api/*]            []               [get]
                                                [/api]              []               [get]
                                                [/apis/*]           []               [get]
                                                [/apis]             []               [get]
                                                [/healthz]          []               [get]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/livez]            []               [get]
                                                [/openapi/*]        []               [get]
                                                [/openapi]          []               [get]
                                                [/readyz]           []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
                                                [/version]          []               [get]
```

These permissions are the base ones assigned to the `system:authenticated` group, as I've not made any specific rights assignments to `jane`. Note there's a *Warning* at the top which lets us know that webhook authorization is not covered here. This is kind of important, because it means that if we're auditing permissions on a cluster, we can't rely on the output of this command to include all of a users rights.

Next up, let's try something different, we'll ask `kubectl` if `jane` can get pods.

```
 kubectl --kubeconfig=jane.kubeconfig auth can-i get pods
yes
```

and if we look in the webhook's `rights.txt` file we can see that's one of the rights assigned via the webhook.


```
jane:get:pods:default
jane:list:pods:default
```

So that's interesting, `auth can-i` was able to reference permissions granted via the webhook where `auth can-i --list` was not. Why is that? Well (after getting some more information from [SIG-Auth](https://kubernetes.slack.com/archives/C0EN96KUY/p1713697390343119)) I think it works like this.

Basically this difference comes down to whether the Authorization provider implements methods to support listing permissions. RBAC, ABAC and the AlwaysAllow modes of authorization are the only ones to support it, the WebhookAuthorizer does not, so rights granted via that mechanism won't show up in the results of `kubectl auth can-i --list`.

Also, as mentioned in the introduction, admission control can also cause requests to be rejected if they are part of the groups of requests sent to admission control (so any requests which create, update, or delete resources), so even if all authorizers were able to tell you the available permissions that's not a full picture.

## So how do I audit Kubernetes permissions?

All this leads to the question of how you audit permissions in Kubernetes clusters with multiple authorizers. The answer here is that the only way to effectively do it is to review each authorization system that's in place in the cluster, and look at the permissions granted in each one.

This does mean you should be very careful when using automated tooling which audits Kubernetes permissions. In most cases it's quite likely it will only support RBAC, and won't provide any information about rights granted in other authorization systems.

## Conclusion

It's fair to say that the flexibility provided in Kubernetes security model does lead to some complexity, which you need to be aware of when operating or reviewing clusters. In this case it's important to be aware of the limitations of in-built tooling and realise when it's necessary to carry out additional manual reviews.

