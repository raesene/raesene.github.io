---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 7 - Container Orchestration Tool Auditing" 
date: 2022-11-12 11:27:00 +0100
comments: false
categories: Kubernetes
---

This is the seventh part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at what the document calls Container Orchestration Tool Auditing, which for this blog will focus on the [Kubernetes auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) feature. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

When looking at Kubernetes auditing the first thing to check is, whether it's enabled or not. The default is not to enable auditing in base Kubernetes, so it requires cluster operators to either configure it directly on the API server (for unmanaged distributions), or to check whether it's enabled via their Cloud Service Provider interface (for managed distributions).

The next thing to investigate is, what exactly is going to be audited. In unmanaged clusters, the operator has flexibility to defined exactly what it is they're going to audit. Kubernetes auditing feature is pretty flexible, allowing for different operations to be captured at different levels or indeed to explicitly avoid capturing specific activity that might be noisy and not interesting from a security perspective. What most policies do have specific activities either captured or blocked and then have a catch-all at the end to handle any cases that are not specifically handled. 

In managed clusters, typically there is one audit policy, which is the one defined by the CSP and which can't be changed. It can be a bit tricky to find out exactly what the setting is, as it's often not well documented. My current best guess for the three major providers is below (any pointers on better sources appreciated :))

- [AKS](https://github.com/Azure/aks-engine/blob/master/parts/k8s/addons/audit-policy.yaml)
- [EKS](https://aws.github.io/aws-eks-best-practices/security/docs/detective/)
- [GKE](https://github.com/kubernetes/kubernetes/blob/release-1.10/cluster/gce/gci/configure-helper.sh#L706)

When reviewing an audit policy, it's worth understanding what's captured at the different levels of auditing. Generally metadata will capture a lot of the information that you might want to review, like the user making the request and the URL of the API endpoint that'll tell you what they did. In some cases you might want to capture the full request, which has more details of exactly what was done.

Another point to note when looking at Kubernetes auditing is that there is a limitation if you're trying to track down whether a specific user carried out a specific action (not an uncommon scenario). The Audit log doesn't capture the source of the credential used to authenticate to the cluster, so if there are multiple credentials for a given user (quite likely if an attacker has access to the CertificateSigningRequest or TokenRequest APIs) there's no easy way to tell if it was really that user, or a cloned credential.

In terms of specific recommendations from the PCI guidance document, there's just the one.

## Section 7.1

**Threat** - Existing inventory management and logging solutions may not suffice due to the ephemeral nature of containers and container orchestration tools integration.

**Best Practice** - Access to the orchestration system API(s) should be audited and monitored for indications of unauthorized access. Audit logs
should be securely stored on a centralized system.

**Details** - The goal of this recommendation is really to say that you need specific auditing that understands containers and container orchestration, which for Kubernetes is going to be the auditing feature. The second part of this recommendation is a pretty standard piece of good practice which is that you shouldn't only store the audit logs on the cluster servers, but instead ensure that their securely stored on a centralized system so that, if an attacker compromises a cluster control plane node, you don't risk them corrupting the audit logs themselves. 

# Conclusion

Auditing is a foundational detective control and fortunately Kubernetes has a well developed auditing feature, which can capture important security activities if correctly configured. Next time we'll be looking at [Monitoring](https://raesene.github.io/blog/2022/11/19/PCI-Kubernetes-Section8-monitoring/)