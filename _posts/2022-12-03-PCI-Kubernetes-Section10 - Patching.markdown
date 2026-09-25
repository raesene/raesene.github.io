---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 10 - Patching" 
date: 2022-12-03 15:27:00 +0100
comments: false
categories: Kubernetes
---

This is the tenth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at Patching. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html)..

Whilst patching is a common part of the security landscape, there are a couple of specific considerations when applying it containerized environments. 

The first one is around patching applications running in containers. As the containers themselves are ephemeral, it's not advisable to patch running instances. Instead the image that the container is based on needs to be patched, that new image needs to be pushed to a container registry and then new instances of the containers deployed to the Kubernetes cluster.

The second consideration is patching Kubernetes itself. The open source project has a policy to provide patches for the current version and previous two released versions (and then provide patches for up to two months after that initial support period has ended). However, most cluster operators to not use Kubernetes directly, instead they make use of one of the many Kubernetes distributions. The support policy for these distributions will vary, although in general they don't provide a huge amount of additional support over the base level of support provided by the Kubernetes project itself.

The recent [Datadog container survey](https://www.datadoghq.com/container-report/) noted that quite a lot of clusters are not running on the latest version of Kubernetes, indeed the most deployed version at the time of the survey was 1.21, despite 1.24 being available to install.

The last thing to note about patching Kubernetes environments is the importance of patching the underlying cluster nodes. This can often be overlooked as nodes tend to be a less visible part of the cluster, and often don't go through the same CI/CD process as containers do. It is especially important that the operating system kernel and CRI components (e.g. Containerd or CRI-O) are patched regularly, as a missing patch could lead to a container breakout. On that note the Datadog survey did note that 30% of cluster nodes using Containerd were running an unsupported version, indicating that this is an area that needs to be addressed.

In terms of the PCI requirements there's three in this section.

## Section 10.1

**Threat** - Outdated container orchestration tool components can be vulnerable to exploits that allow for the compromise of the installed cluster or workloads.

**Best Practice** - All container orchestration tools should be supported and receive regular security patches, either from the core project or back-ported by the orchestration system vendor.

**Details** - For this requirement, it's important to find out the support lifecycle of the software in use, there's a note of some common ones for Kubernetes distributions [here](https://www.container-security.site/general_information/support_lifecycles.html). 

## Section 10.2

**Threat** - Vulnerabilities present on container orchestration tool hosts (commonly Linux VMs) will allow for compromise of container orchestration tools and other components.

**Best Practice** - Host operating system of all the nodes that are part of a cluster controlled by a container orchestration tool should be patched and kept up to date. With the ability to reschedule workloads dynamically, each node can be patched one at a time, without a maintenance window.

**Details** - In addition to making sure that operating system patches are applied, it's important that where a kernel security patch has been applied, the node(s) in question have been rebooted such that the updated kernel is in use (unless hot-patching techniques are being used).

## Section 10.3

**Threat** - As container orchestration tools commonly run as containers in the clusters, any container with vulnerabilities may allow compromise of container orchestration tools.   

**Best Practice** - All container images used for applications running in the cluster should be regularly scanned for vulnerabilities, patches should be regularly applied, and the patched images redeployed to the cluster.

**Details** - Reviewing a cluster for this best practice can be achieved using container scanning tools like [Trivy](https://github.com/aquasecurity/trivy) or [grype](https://github.com/anchore/grype).

# Conclusion

Like many of the sections of the PCI guidance the topic in question is fairly common good practice, however as we've discussed there are a couple of specific considerations when applying it to Kubernetes environments.
