---
layout: post
title: "Using the API server proxy to bypass network policies"
date: 2026-08-21 10:00:00 +0100
comments: false
categories: Kubernetes
---

I've been doing some work for an upcoming talk on Kubernetes Multi-tenancy security, and as part of that I was thinking about one of my favourite topics in Kubernetes security, [SSRF](https://www.container-security.site/attackers/kubernetes_ssrf.html). As I was doing that I realised that applying a known existing weakness in Kubernetes security could be used by attackers to get unauthorised to other tenant's workloads in a multi-tenant cluster.

In 2019 Kinvolk published a blog on [abusing Kubernetes API server proxying](https://kinvolk.io/blog/2019/02/abusing-kubernetes-apiserver-proxying) which talked about the idea of overwriting a pod's IP address to allow access to an external IP address with the API server's network position, and it occurred to me that is likely to apply to internal IP addresses too. The mitigating discussed in that blog which was placing the API server in the same network position as worker nodes, made sense when the attacker was trying to access IP addresses external to the cluster, but doesn't really apply to an attacker who's trying to get access to an internal resource that should be blocked.

## Setup

So imagine we have a simple Multi-tenant cluster, where each tenant has control over the workloads in their namespace, but shouldn't have access to any other workloads in the cluster. We use network policies to implement this restriction blocking access from each namespace, to any other namespace. We do however allow the API server itself to access every namespace for cluster management and to allow tenants to debug their own applications. It might look something like :-

![Multi-tenant cluster seutp]({{site.url }}/assets/media/multi-tenant-cluster-network.png)

With this setup, we can connect to our Pod via the API server proxy allowing each tenant to work with their own applications

![Multi-tenant cluster access via API server proxy]({{site.url }}/assets/media/multi-tenant-cluster-network-access-allowed.png)

However as each tenant only has permissions to its own namespace, they can't use the API server proxy to access workloads in the other tenant's namespace.

## The Attack

As mentioned in the Kinvolk blog post if a user has rights to overwrite the status of a pod, they can change its IP address. When that happens the cluster will change it back, but if we loop the change it'll be present long enough to (ab)use!

So assuming that one tenant can work out what IP address they want to connect to in another tenant's namespace (this might get disclosed by a number of methods for example [DNS bruteforcing](https://github.com/jpts/coredns-enum)) they can set their own pod's IP address to that one and connect via the API server!

This gets allowed by RBAC as the proxy request is to a pod that the user owns and is allowed by network policies as the API server has access to all tenant's namespaces.

In the demonstration video below I've setup a cluster with two web applications and network policies blocking access from one namespace to the other. Then running the `kubectl proxy` command as the tenant-a-user it shows that they can get to their own application via the API server proxy but not the one from tenant-b. Then we launch the attack script that sets the IP address of the tenant-a pod to be the address of tenant-b's pod, after which it's possible to get access to tenant-b's application via the proxy.

<iframe width="560" height="315" src="https://www.youtube.com/embed/MMOdag1ouTA?si=Q0WGwAGv0TFsc-ym" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Conclusion

Kubernetes network security is a kind of complex topic as things like the [unpatchable four](https://securitylabs.datadoghq.com/articles/?s=unpatchable) have shown and in a multi-tenant environment that gets even more complex!

## Appendix A - Reproducing the attack

