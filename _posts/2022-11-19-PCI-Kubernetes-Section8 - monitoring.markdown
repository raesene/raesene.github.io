---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 8 - Container Monitoring" 
date: 2022-11-19 11:27:00 +0100
comments: false
categories: Kubernetes
---

This is the eighth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at Container Monitoring, which follows on from the last part about [Container orchestration tool auditing](https://raesene.github.io/blog/2022/11/12/PCI-Kubernetes-Section7-Auditing/). An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

Container monitoring is another topic where it's important to consider the way that Kubernetes clusters operate, as standard approaches to monitoring might not be sufficient. There's a couple of properties to consider when designing container monitoring.

1. Containers are ephemeral - Containers running in a Kubernetes cluster can be moved around by automated processes like the Kubernetes scheduler, to ensure the smooth running of the environment. This means that any local monitoring on a cluster node is unlikely to capture all relevant logs. It also means that logging must be centralised so that all logs relating to an application running in containers can be queried from one place (operators connecting to every node in a cluster to look for container logs would *not* be a sensible solution)

2. Whilst containers are generally (but not always) Linux processes, you can't really rely on host level monitoring when running applications in containers as, at a host level, there isn't sufficient context to make sensible security decisions. To give a specific example, imagine a host level security monitoring tool sees suspicious behaviour in the namespace of an `nginx` webserver. Whilst it can report that, it doesn't know that the `nginx` process in question belongs to a specific Kubernetes pod, in a specific Kubernetes Deployment, in a specific Kubernetes namespace, that's owned by a specific team in the company. All of that context is really needed to ensure that teams can find and react to security issues quickly.

The result of this is that any security monitoring system for a Kubernetes cluster has to be Kubernetes aware and centralized to be really effective, and when reviewing the security of in-scope clusters, it's important to ensure that this is in place.

In addition to capturing container logs, it's also important to have monitoring in place that can detect attempts by attackers to compromise containers or breakout to the underlying host. Again it's important that this tooling is container aware (for example understands how namespaces are used in containers) to be really effective.

In terms of the PCI requirements there's two in this section.

## Section 8.1

**Threat** - Local logging solutions will not allow for appropriate correlation of security events where containers are regularly destroyed.

**Best Practice** - Centralized logging of container activity should be implemented and allow for correlation of events across instances of the same container

**Details** - On Kubernetes there are a wide range of approaches to centralized container monitoring. Generally logs from individual cluster nodes will be transferred to a cloud hosted or on-premises service, with sufficient information to ensure that it's possible to correlate which Kubernetes resource they belonged to.

## Section 8.2

**Threat** - Without appropriate detection facilities, the ephemeral nature of containers may allow attackers to execute attacks unnoticed.

**Best Practice** - Controls should be implemented to detect the adding and execution of new binaries and unauthorized modification of container files to running containers.

**Details** - For Kubernetes this will typically mean deploying an open source or commercial container runtime security product. These products should be designed to capture attacks occurring in Kubernetes containers and as with the log monitoring, store them in a centralized location. Typically these products will have a ruleset allowing for detection of common container attacks, and ideally should allow for custom rules to be added by the cluster operator to reflect specific risks in their environment.

# Conclusion

Container monitoring is another area where the controls we need are similar to non-containerized environments but, due to the nature of container based architectures, it's important to ensure that the deployed solutions take account of their environment. The use of Kubernetes specifically may not affect how those tools operate, but it's important that the tools are Kubernetes aware so that they can provide all the relevant information to operators about possible attacks occurring in monitored clusters. Next time we'll be looking at [Runtime Security](https://raesene.github.io/blog/2022/11/27/PCI-Kubernetes-Section9-Runtime-Security/).