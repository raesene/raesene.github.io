---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 16 - Segmentation" 
date: 2022-12-20 9:27:00 +0100
comments: false
categories: Kubernetes
---

This is the sixteenth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at the Segmentation section. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

The topic of segmentation in Kubernetes is an interesting one. First let's talk a bit about what PCI means by Segmentation. From [this document](https://listings.pcisecuritystandards.org/documents/Guidance-PCI-DSS-Scoping-and-Segmentation_v1.pdf) we can see this definition

> Segmentation involves the implementation of additional controls to separate systems with different security needs. For example, in order to reduce the number of systems in scope for PCI DSS, segmentation may be used to keep in-scope systems separated from out-of-scope systems. 
> Segmentation can consist of logical controls, physical controls, or a combination of both. Examples of commonly used segmentation methods for purposes of reducing PCI DSS scope include firewalls and router configurations to prevent traffic passing between out-of-scope networks and the CDE, network configurations that prevent communications between different systems and/or subnets, and physical access controls.

So in order to implement segmentation in a containerized environment we need to put controls in place so that there is effective security segregation between in-scope workloads and out-of-scope workloads.

With Kubernetes there's a couple of ways you can implement this kind of control. The easiest (from a security point of view) is to use separate clusters for in-scope and out-of-scope workloads, however some organizations might not like this approach as it reduces the cost benefits of Kubernetes as it requires multiple sets of control plane nodes and reduces the ability to share resources between workloads.

The other approach is to try and use a single cluster for both in-scope and out-of-scope workloads, we need to harden the cluster such that we're providing appropriate security segmentation, a.k.a hard multi-tenancy.

## Hard Multi-Tenancy in Kubernetes

To provide hard multi-tenancy in a Kubernetes cluster there are a number of considerations that need to be taken into account, and challenges to be overcome. Typically this kind of solution would be based on the use of Kubernetes namespaces as a unit of security segmentation, but it's important to recognize that this (and Kubernetes in general) wasn't designed for a hard multi-tenancy use case.

The sections below aren't intended to be an exhaustive treatment of the challenges of hard multi-tenancy in Kubernetes (that would require it's own blog post series!) but to indicate some of the complexity and why it's not a trivial problem to solve.

### Kubernetes API Segregation

The first challenge is that the Kubernetes API itself. There are a number of resources in a cluster wide and not namespaced, so we need to ensure that users in the "low security" namespace(s) can't access these resources, which restricts the facilities that they can use. This particular issue can be mitigated via the use of "virtual cluster" style solutions such as [vcluster](https://www.vcluster.com/), which create virtual Kubernetes clusters on top of a single host cluster. You can then provide full access to the Kubernetes API to the virtual cluster, but restrict access to the host cluster.

If you're not using a virtual cluster solution, part of this also involves strict RBAC controls which prevent "low security" users from escalating their rights to access "high security" workloads. There's a page on the [Kubernetes site](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#privilege-escalation-risks) which discusses some of the areas to consider here.

### Workload Segregation

Virtual clusters alone, however, don't provide the full solution. Where workloads are being deployed to a shared set of clusters nodes there is a risk that any workload can break out to an underlying node from the "low security" namespace and then access parts of the "high security" environment. Mitigating this will require adoption of admission control solutions such as [Kyverno](https://kyverno.io/) with a highly restrictive set of policies to reduce the risk of privilege escalation. Typically you'd expect these policies to be in-line with the [restricted PSS](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) policy.

This doesn't provide a complete picture, however as you still have the risk of container breakout via Linux kernel/runc/Containerd/Docker CVEs. You can reduce this risk by using a solutions like [gVisor](https://gvisor.dev/) or [Kata Containers](https://katacontainers.io/) to provide a smaller attack surface, hardening the container runtime environment.

Another approach which might help here is to implement separate node pools for each environment. This reduces the workload resource sharing benefit of Kubernetes, but does reduce the risk of a breakout from one environment to another. 

There is also a complication with this approach, which is that any workloads which have privileged access to the Kubernetes API server (e.g. operators, or admission control services) should not be placed in the "low security" node pool, as this would allow them to escalate privileges to the "high security" environment, via service account tokens. This approach also relies on the use of "node authorization" in Kubernetes otherwise the Kubelet credentials can be used to escalate privileges to the "high security" environment. Whilst this plugin is enabled in most Kubernetes distributions, it's not guaranteed, (for example at the time of writing it's not enabled in AKS and cluster operators cannot enable it by themselves).

### Network Segregation

As we discussed back in the [network section](https://raesene.github.io/blog/2022/10/23/PCI-Kubernetes-Section4-network-security/) Kubernetes defaults to an open flat network for all workloads in the cluster. This is obviously not suitable for a hard multi-tenancy solution, so it would be necessary to implement strict network policies restricting traffic between the two environments. 

However there's another aspect of network segregation in Kubernetes which can be tricky to mitigate, which is DNS. DNS is used for service discovery in clusters, and this is a cluster-wide service. To provide effective segregation it would be necessary to split the DNS service into two separate services, one for each environment. Without this it's generally trivial for an attacker to enumerate every service in the cluster, using commands like `dig +short srv any.any.svc.cluster.local`.

In terms of the PCI requirements there's four in this section.

## Section 16.1

**Threat** -  Unless an orchestration system is specifically designed for secure multi-tenancy, a shared mixed-security environment may allow attackers to move from a low-security to a high-security environment.

**Best Practice** - Where practical, higher security components should be placed on dedicated clusters. Where this is not possible, care should be taken to ensure complete segregation between workloads of different security levels

**Details** - Reviewing an environment for this requirement, would generally involve looking at deployed workloads for the in-scope clusters and confirming that they are only running in-scope workloads.

## Section 16.2

**Threat** -  Placing critical systems on the same nodes as general application containers may allow attackers to disrupt the security of the cluster through the use of shared resources on the container cluster node.

**Best Practice** - Critical systems should run on dedicated nodes in any container orchestration cluster.

**Details** - As discussed using dedicated node pools is an option to try and ensure workload segregation, but it's a tricky one to implement well. Reviewing a cluster for this would generally involve looking at the worklods deployed to each environment and confirming that there are no privilege escalation paths, via things like service account tokens, or Kubelet credentials.

## Section 16.3

**Threat** -  Placing workloads with different security requirements on the same cluster nodes may allow attackers to gain unauthorized access to high security environments via breakout to the underlying node.

**Best Practice** - Split cluster node pools should be enforced such that a cluster user of the low-security applications cannot schedule workloads to the high-security nodes. 

**Details** - For workload scheduling segregation, admission control solutions are required, so reviewing an environment for this would involve reviewing the policies in place and also reviewing the security of the admission control solution itself.

## Section 16.4

**Threat** - Modification of shared cluster resources by users with access to individual applications could result in unauthorized access to sensitive shared resources.

**Best Practice** - Workloads and users who manage individual applications running under the orchestration system should not have the rights to modify shared cluster resources, or any resources used by another application.

**Details** - How this is reviewed would depend on the approach taken. Where a virtual cluster solution is used, it might be possible to review to ensure that virtual cluster admins have no access to the underlying master cluster. Where Kubernetes RBAC is used for this, it would require a review of RBAC policies to ensure that users can't escalate privileges to access in-scope workloads from the "low security" environment.

# Conclusion

Segmentation is an important part of PCI security and applying it to Kubernetes can be tricky, as it's not designed for hard multi-tenancy. 

