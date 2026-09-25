---
layout: post
title: "Some notes on Kubernetes Network Policies"
date: 2018-03-25 15:00:39 +0100
comments: false
categories: Kubernetes
---

Kubernetes [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) are a useful security feature which allow for traffic into and (sometimes) out of pods to be restricted.

This is very useful if you want to add another layer of defence to your cluster and reduce the risk of attacks both on other services running on the cluster and also the control plane services like etcd and the Kubelet.

To make use of network policies, you need to have a k8s version that supports them (the Network Policy API hit stable in [1.7](http://blog.kubernetes.io/2017/06/kubernetes-1.7-security-hardening-stateful-application-extensibility-updates.html)).  You'll also need a network plugin that supports Network Policies, in order for your policies to be effective.  Most of the major network plugins support network policies, however there are some irregularities to be aware of.

For example Weave doesn't currently support egress policies at the moment (issue [here](https://github.com/weaveworks/weave/issues/2624) ), so do check the support of your chosen plugin before starting :)

## Network Policy Concepts

Network policies are a lot like network ACLs or firewall rules, if you're familiar with those.  Without any network policies in effect on a set of pods, there's a "default allow" rule in place. However as soon as any network Policy applies to a given pod, that pod has a "default deny" setup applied, meaning you have to specify all the traffic desired for that pod once you've started implementing network policies on it.

There are two types of network policies that can be specified.  Ingress policies restrict traffic to set of pods, and egress policies restrict outbound traffic from a set of pods.

## Practical examples

The Kubernetes documentation on [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) has some good examples of policies you might want to apply, and there's also a [repo. on github](https://github.com/ahmetb/kubernetes-network-policy-recipes) with some more examples, with a nice visualization of the effect of the policy, however lets cover one example that's not covered in either of those resources.

## Denying access to Kubernetes nodes

One of the challenges I've seen in assessments of Kubernetes cluster security, when we work from a "compromised container" perspective, is that it's possible to attack the underlying nodes and this exposes the control plane services to attack.  It's not uncommon to see unauthenticated access to etcd or the Kubelet, and unauthorised access to those services is pretty bad for the overall security of the cluster.

So can we use network policies to prevent this?  The answer seems to be "yes, but with some side effects".

### Test cluster setup

I've got a demo 3 node 1.9 cluster using Calico for the network plugin.  The host network is on `192.168.111.0/24` so the goal of my network policy is to restrict access to that network, whilst still allowing the pods to communicate with the rest of the world.

I've got a namespace setup for this test (netpol-test) and a sample alpine container running there, so we can check the results of our network policies.

### Basic egress restricting policy

So at a basic level we can use something like this

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-block
  namespace: netpol-test
spec:
  podSelector: {}
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 192.168.111.0/24
  policyTypes:
  - Egress
```

to restrict outbound access.  What you can see here is that we're applying the network policy to a specific namespace with `namespace: netpol-test` and then we're using a blank pod selector `podSelector: {}` to hit all the pods in that namespace.

Then we're specifying an egress policy.  Network policies are all about allowing traffic after the initial "default deny" is in place, so we have to specify our restriction in a kind of back-handed way.

We allow all destinations with `cidr: 0.0.0.0/0` and then block a specific network with the `except:` block.

If you look at the effects before and after using something like nmap, you'll see that ports like `10250/TCP` which were accessible before applying the policy, are no longer visible afterwards.

So what about the side effects that I mentioned above?  Well when I was testing this network policy, I noticed that in addition to blocking access to the nodes directly (on 192.168.111.0/24) this policy also prevents access to the Kubernetes service, which in the case of this cluster is on 10.96.0.1:443.  

This could be desirable from a security standpoint, as it's blocking access to control plane services, but it is rather un-intuitive to have a completely different network blocked based on a network policy.

### Allowing access to a service

So in this model where we're using network policies to restrict control plane service access, the one service we may want to allow containers to speak to is the Kubernetes API.  As the access provided by network policies is cumulative, this is pretty straightforward.  Working with the same setup as the previous example, we can just apply something like this

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-api
  namespace: netpol-test
spec:
  podSelector: {}
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.111.0/24
    ports:
    - protocol: TCP
      port: 6443
  policyTypes:
  - Egress
```

and we'll have access to the API service from pods in the `netpol-test` namespace.  Also our oddness from the previous example is present here as well.  Applying this policy also opens up access to 10.96.0.1:443 even though that's not the port or IP address specified!

## Conclusion

This was just a brief introduction to Network policies in Kubernetes, there's a lot more that can be done with them in terms of allowing access between specific pods and services.  Overall they're a good layer of additional protection, and one that's well worth considering for production clusters, although I have a feeling that if used heavily, management of the rulesets could get a bit complex...