---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 15 - Configuration Management" 
date: 2022-12-18 15:27:00 +0100
comments: false
categories: Kubernetes
---

This is the fifteenth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at the Configuration Management section. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

Similarly to the previous section on [version management](https://raesene.github.io/blog/2022/12/16/PCI-Kubernetes-Section14-Version-Management/) this section isn't so much around security configuration as the policies and processes that should be in place in an in-scope environment.

As the guidance here relates to the secure configuration of container orchestration environments, and having companies develop standards for secure configuration of their container environments, it's worth noting what options there are for Kubernetes. Whilst companies should customize these to their own environments, there are some options that can be used as a starting point.

It's also worth noting that as described in the post about [the challenges of assessing Kubernetes clusters for PCI Compliance](https://raesene.github.io/blog/2022/09/20/Assessing-Kubernetes-Clusters-for-PCI-Compliance/) one of the things to think about when looking at compliance standards is which distributions and versions are covered.

## CIS Benchmarks

There are a couple of CIS benchmarks which can be relevant to secure configuration of Kubernetes clusters. Firstly there's the [CIS Benchmark for Docker](https://www.cisecurity.org/benchmark/docker). It's important to note here that this benchmark was designed for standalone Docker installations and not for a use case where Docker is a Container Runtime in a Kubernetes cluster. A number of the requirements will not apply to Kubernetes environments that use Docker. It's also worth noting that there are no current benchmarks for Containerd or CRI-O.

At the Kubernetes level there is a set of related CIS Benchmarks. The top-level CIS [Benchmark for Kubernetes](https://www.cisecurity.org/benchmark/kubernetes) is designed to address [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) clusters. Using it for any other Kubernetes distribution could require the details of checks to be modified to take account of differences in the implementation of Kubernetes. There are versions of the benchmark for various versions of Kubernetes going back to 1.16, however not every version of k8s has a corresponding CIS benchmark.

There are also a set of CIS Benchmarks for various managed Kubernetes distributions, which provide more specific detail for those environments. C Currently there is coverage for [GKE](https://cloud.google.com/kubernetes-engine), [EKS](https://aws.amazon.com/eks/), [AKS](https://azure.microsoft.com/en-gb/products/kubernetes-service/), [ACK](https://www.alibabacloud.com/product/kubernetes), [OKE](https://www.oracle.com/uk/cloud/cloud-native/container-engine-kubernetes/) and [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift)

## NSA Kubernetes Hardening Guide

Whilst it often gets used similarly to the CIS benchmarks the [NSA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF) is wider in scope and not designed to be a configuration standard. Whilst it does provide some guidance on specific settings in some areas, it doesn't set out to be detailed across all areas, in the way the CIS benchmarks do. 

## DISA STIG

The DISA [Kubernetes Security Technical Implementation Guide](https://www.stigviewer.com/stig/kubernetes/2021-04-14/) is another option which does have detailed requirements. However it's important to note that, as of December 2022, it's not been updated recently, so checks may be outdated and whilst it doesn't specify a specific Kubernetes distribution it addresses, from the paths in the document it appears to be Kubeadm.

In terms of the PCI requirements there's one in this section.

## Section 15.1

**Threat** -  Container orchestration tools may be misconfigured and introduce security vulnerabilities.

**Best Practice** 
- a. All configurations and container images should be tested in a production-like environment prior to deployment.

- b. Configuration standards that address all known security vulnerabilities and are consistent with industry-accepted hardening standards and  vendor security guidance should be developed for all system components, including container orchestration tools.
  - i. Address all known security vulnerabilities.
  - ii. Be consistent with industry-accepted system hardening standards or vendor hardening recommendations.
  - iii. Be updated as new vulnerability issues are identified.

**Details** - Assessing the first part of this recommendation would involve understanding the companies CI/CD environment and how changes to cluster configuration are tested. Typically some form of automated testing should be done on any Kubernetes manifest before deployment. For the second part understanding which standard(s) are in use and how compliance is achieved (e.g. CSPM tooling)

# Conclusion

Creating a security configuration standard is a sensible part of larger Kubernetes deployments as it will help to maintain consistent configuration and security levels across the environment, however it does require some effort to tailor this to specific Kubernetes distributions and versions.