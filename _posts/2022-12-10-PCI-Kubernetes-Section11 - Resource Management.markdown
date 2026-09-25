---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 11 - Resource Management" 
date: 2022-12-10 08:27:00 +0100
comments: false
categories: Kubernetes
---

This is the eleventh part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at Resource Management. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

When looking at resource management in containerized environments, the first thing to consider is that because most containers are just processes running on a shared kernel, there is a risk that a single container could consume all of the resources available on a node. Also, unlike virtual machine based environments, by default there are no resource constraints in place.

As a result, it's important to define limits on obvious resources like CPU and memory and also things like processes (to stop a fork-bomb in a container from taking down the node). Container tooling will let you define these limits and, under the covers, Linux cgroups will be used to enforce them.

Within Kubernetes there are a number of ways to define resource limits, workloads should define their own requests for resources in [their spec](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) and then these are combined with [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) objects defined at a namespace level. 

Having this information in place will help the Kubernetes scheduler to make better decisions about where to place workloads, and also to ensure that the workloads don't consume more resources than they are allowed to.

In terms of the PCI requirements there's just one in this section.

## Section 11.1

**Threat** - A compromised container could disrupt the operation of applications due to excessive use of shared resources.

**Best Practice** - All workloads running via a container orchestration system should have defined resource limits to reduce the risk of “noisy neighbors” causing availability issues with workloads in the same cluster.

**Details** -  To assess whether this is in place, an assessor would look for `ResourceQuota` objects in the cluster, and also for resource limits defined in the workload specs.


# Conclusion

Resource management in containerized environments is something which requires particular attention, when compared to using virtual machines or physical servers as the default position of having no constraints and the shared kernel means that a single container can consume all of the resources available on a node. This is why it's important to define resource limits for all containers, and to ensure that the limits are enforced.
