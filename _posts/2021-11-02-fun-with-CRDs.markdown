---
layout: post
title: "Fun with CRDs - Overwriting core types" 
date: 2021-11-01 09:50:39 +0100
comments: false
categories: Kubernetes
---

I was playing around with CRDs today when I wondered "what would happen if I tried to overwrite one of Kubernetes' core object types? The results weren't what I expected, so thought it was worth writing it down, in case it comes as a surprise to others :) **N.B. There's an update at the bottom which sheds more light on what happens when you do this**

## What's a CRD?

Kubernetes Custom Resources are a way to extend the core Kubernetes API with custom types. They're heavily used by various programs which extend Kubernetes functionality, and a common feature of most clusters.  In general, Custom Resources have their own group so, for example if you use Cilium for your kubernetes networking, you'll have CRDs like `ciliumendpoints.cilium.io` which is the ciliumendpoint object in the cilium.io group.

## Overwriting core Kubernetes CRDs

So an obvious question for a curious goose is, what happens if I try to create a CRD with the `k8s.io` group, which is used by a lot of core Kubernetes objects. Well, let's find out !

If we write a basic CRD which starts like this

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: networkpolicies.networking.k8s.io
```

we get an error when we create it. The error handily tells us that we need an annotation added to create a CRD in this group

```
The CustomResourceDefinition "networkpolicies.networking.k8s.io" is invalid: metadata.annotations[api-approved.kubernetes.io]: Required value: protected groups must have approval annotation "api-approved.kubernetes.io", see https://github.com/kubernetes/enhancements/pull/1111
```

There's no validation on the contents of this annotation, so I just took the sample one from the GH issue. After I did that, it works! The first time I created a CRD overwriting the Network Policy object, I had it as a namespace scoped object and the results were unexpected.

Checking `kubectl get crds` shows that it created ok (The one I created didn't have an egress field)

```
kubectl get crds
NAME                                CREATED AT
networkpolicies.networking.k8s.io   2021-11-01T19:28:04Z
```

but if I use `kubectl explain` on the object, the definition I get back is the original core object.

```
kubectl explain networkpolicies.networking.k8s.io.spec
KIND:     NetworkPolicy
VERSION:  networking.k8s.io/v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior for this NetworkPolicy.

     NetworkPolicySpec provides the specification of a NetworkPolicy

FIELDS:
   egress       <[]Object>
     List of egress rules to be applied to the selected pods. Outgoing traffic
     is allowed if there are no NetworkPolicies selecting the pod (and cluster
     policy otherwise allows the traffic), OR if the traffic matches at least
     one egress rule across all of the NetworkPolicy objects whose podSelector
     matches the pod. If this field is empty then this NetworkPolicy limits all
     outgoing traffic (and serves solely to ensure that the pods it selects are
     isolated by default). This field is beta-level in 1.8
```

Also if I try to create a sample network policy which uses the original object type, it works ok. So the CRD is there, but it's not working. 

What happens if you change the scope in the CRD from namespaced to cluster? Well now, it shows in the CRD list *and* it's overwritten the object definition retrieved by `kubectl explain`

```
kubectl explain networkpolicies.networking.k8s.io.spec
KIND:     NetworkPolicy
VERSION:  networking.k8s.io/v1

RESOURCE: spec <Object>

DESCRIPTION:
     <empty>

FIELDS:
   name <string>
```

and if I try to create a Network Policy, I get an error showing that it doesn't understand those field, so this has overwritten the core object.

```
error: error validating "sample-netpol.yaml": error validating data: [ValidationError(NetworkPolicy.spec): unknown field "egress" in io.k8s.networking.v1.NetworkPolicy.spec, ValidationError(NetworkPolicy.spec): unknown field "ingress" in io.k8s.networking.v1.NetworkPolicy.spec, ValidationError(NetworkPolicy.spec): unknown field "podSelector" in io.k8s.networking.v1.NetworkPolicy.spec, ValidationError(NetworkPolicy.spec): unknown field "policyTypes" in io.k8s.networking.v1.NetworkPolicy.spec]; if you choose to ignore these errors, turn validation off with --validate=false
```

## Other examples

There's more prodding at this to be done, but a fun additional one I noticed is that if you try to create a CRD with a name of `users.authentication.k8s.io` it creates, but the result isn't visible in `kubectl api-resources` and you can't use `kubectl explain` on it, even though it's not overwriting a visible Object type.

## Conclusion

From a security standpoint, the takeaway here is, don't let anyone create CRDs in your cluster unless you really really trust them, ~~as they can modify Kubernetes core behaviour~~. Also if you're planning to appear on Rawkode's klustered show, you could try this and hope the cluster fixers don't read my blog :)

## Update 03/11/2021

Curious to get some details on was going on behind the scenes and possible security implications, I asked on k8s slack (a great source of information), and the inimitable [Jordan Liggitt](https://twitter.com/liggitt) was able to explain it. It seems like this doesn't actually overwrite the core Kubernetes types, but what it does do is modify OpenAPI schemes used by clients like Kubectl to validate data being passed to the API server. So, in effect this could break tooling or produce odd behaviour but isn't modifying the core Kubernetes API. There's an issue in the [Kubernetes Github repo](https://github.com/kubernetes/kubernetes/issues/106056) with more info.

To me, it's still interesting, but in a slightly different way as it illuminates how some of the different components in Kubernetes work together and some of the possible complexity when unusual or unexpected changes are made.