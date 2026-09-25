---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 14 - Version Management" 
date: 2022-12-16 15:27:00 +0100
comments: false
categories: Kubernetes
---

This is the fourteenth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at the Version Management section. An index of the posts in this series that I've written so far can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

This one is a slightly interesting one as it's not really a security issue, but more of a best practice, but there are some considerations which are specific to containerization. The main one is the move to "Infrastructure As Code", where the setup of Kubernetes clusters and their applications are stored in formats like [YAML](https://yaml.org/) and [HCL](https://github.com/hashicorp/hcl), and processed by tools like [Helm](https://helm.sh/) and [Terraform](https://www.terraform.io/). Having all of this information stored in files lends itself to improved version management practices as, in theory anyway, everything which is neeeded to re-create the environment is stored in the files.

Of course, in addition to the tools, we need an approach to managing their versioning and storage if we want to achieve our goals. One approach which lends itself to this is [gitops](https://www.weave.works/technologies/gitops/) which uses git as the source of truth for the environment. 

These tools and approaches are not specific to Kubernetes, but they are a natural fit for it, and so it's worth considering them when looking at the PCI requirements.

In terms of the PCI requirements there's one in this section.

## Section 14.1

**Threat** - Without proper control and versioning of container orchestration configuration files, it may be possible for an attacker to make an unauthorized modification to an environment’s setup

**Best Practice** - a. Version control should be used to manage all non-secret configuration files. b. Related objects should be grouped into a
single file. c. Labels should be used to semantically identify objects.

**Details** - Reviewing a cluster for this recommendation would likely start with speaking to the operators to understand how they control the configuration files, and then checking the management of key elements such as the helm charts and terraform files (or alternative tools if they are being used).

# Conclusion

This is a relatively short post as the PCI recommendations are relatively straightforward and Kubernetes doesn't introduce any very specific concerns, however it's another one to add to the list of things to consider.
