---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 9 - Runtime Security" 
date: 2022-11-27 11:27:00 +0100
comments: false
categories: Kubernetes
---

This is the ninth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at Runtime Security. This section works in conjunction with the one on [workload security](https://raesene.github.io/blog/2022/10/15/PCI-Kubernetes-Section3-workload-security/). Where that one looked at restricting the rights that ordinary containers have to underlying nodes, this section is more about when you might want to look at alternatives to standard "docker style" containers. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

This comes down to the long-standing question in container security "do containers contain". In my opinion there's no binary answer to this question. Linux containers do provide a level of isolation to the underlying node *but* there is a very large attack surface, and as we've seen this year there have been a number of Linux kernel vulnerabilities which have turned into [container breakout attacks](https://www.container-security.site/attackers/container_breakout_vulnerabilities.html).

There are way to use container style workflows and Kubernetes while improving the level of isolation provided. Typically the trade-off is that improved isolation has a level of performance impact, and sometimes workloads need more access to the host than can be provided.

Some approaches to addressing this problem include [gVisor](https://gvisor.dev/) and [katacontainers](https://katacontainers.io/). Additionally, another approach which could work where you have need higher levels of workload isolation, is to use a "serverless" style approach with something like AWS Fargate, working with an EKS cluster, as here there should be no user accessible underlying node to be accessed, and container isolation is handled by the cloud service provider.

Windows containers also face a similar split with "process based" containers and Hyper-V containers being available for use, however there is a challenge when using Kubernetes.

Microsoft specifically state that "[Only hypervisor-isolated containers provide a security boundary, and process-isolated containers do not](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-security#container-security-servicing-criteria)", so where a security boundary is required, Hyper-V containers must be used.

In Kubernetes however [Hyper-V containers are not supported](https://kubernetes.io/docs/concepts/windows/intro/#windows-nodes-in-kubernetes), so unless a serverless container option is available, high-risk workloads need to be run on a separate cluster.

In terms of the PCI requirements there's two in this section.

## Section 9.1

**Threat** - The default security posture of Linux process-based containers provides a large attack surface using a shared Linux kernel. Without hardening, it may be susceptible to exploits that allow for container escape.

**Best Practice** - Where high-risk workloads are identified, consideration should be given to using either container runtimes that provide hypervisor-level isolation for the workload or dedicated security sandboxes.

**Details** - Once "high-risk" workloads have been identified, it should be possible to check whether the Kubernetes clusters in use are making appropriate use of additional isolation techniques. Typically to implement something like gVisor the workloads would specify the runtime needed using an annotation in the manifest (`runtimeClassName: gvisor`). Alternatively if managed Kubernetes is in use, checking to see if a serverless container style was in use could be carried out.

## Section 9.2

**Threat** - Windows process-based containers do not provide a security barrier (per Microsoft’s guidance) allowing for possible container break-out.

**Best Practice** - Where Windows containers are used to run application containers, Hyper-V isolation should be deployed in-line with Microsoft’s security guidance.

**Details** - As mentioned above for Kubernetes if high-risk containers are being run in Windows nodes, either a serverless option or separate clusters would be needed.

# Conclusion

The issue of exactly how much security isolation is provided by Linux containers is a somewhat tricky question. It's clear that they do provide some level of isolation, but with the attack surface of the Linux kernel and the other parts of the software stack, the isolation may not be sufficient for higher risk workloads. Fortunately there are other options which can be deployed with Kubernetes to provide additional isolation where needed. Next time we'll be looking at [Patching](https://raesene.github.io/blog/2022/12/03/PCI-Kubernetes-Section10-Patching/).