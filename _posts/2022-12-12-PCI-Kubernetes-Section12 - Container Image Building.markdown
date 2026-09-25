---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 12 - Container Image Building" 
date: 2022-12-12 15:27:00 +0100
comments: false
categories: Kubernetes
---

This is the twelfth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at Container Image Building. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

This section overlaps somewhat with [section 3 on workload security](https://raesene.github.io/blog/2022/10/15/PCI-Kubernetes-Section3-workload-security/), but expands it with some specific concerns that need to be addressed about how companies build container images.

Image building is a fundamental part of the container lifecycle, so it's important to understand how to do it securely, and there's a couple of aspects to this.

The first one is the user that the containers run as. By default, Docker and similar tools run containers as the `root` user. Whilst there are some restrictions placed on the container even when it's root, ideally images should be built to run as non-root users and then this restriction should be enforced by Kubernetes admission control. One reason for this is that running as root tends to give a process more access to the Linux kernel meaning that there are more opportunities to exploit Linux kernel vulnerabilities that may exist in the shared kernel of the host.

The second aspect is the base image that is used to build the container image. The base image is the starting point for the image, and it's important to make sure that it's up to date and that it's not been tampered with. This is especially important if the base image is being pulled from a public registry, as there's a risk that it's been compromised. It's also important to make sure that the base image is as minimal as possible, as this reduces the attack surface of the image, and improves maintainability. 

In general, for images an organization are building internally, it's also a good idea to try and standardize on a small set of base images. This eases the burden of keeping the images up to date, and also makes it easier to audit the images to make sure that they're not using any vulnerable packages.


In terms of the PCI requirements there's four in this section.

## Section 12.1

**Threat** - Container base images downloaded from untrusted sources, or which contain unnecessary packages, increase the risk of supply chain attacks

**Best Practice** - Application container images should be built from trusted, up-to-date minimal base images. 

**Details** -  Reviewing a cluster for this requirement is a little tricky, as it's not something that can be easily checked by looking at the cluster. The best way to check for this is to look at the Dockerfiles that are used to build the images, and speak to the developers to understand the process being used to maintain the base images. 

## Section 12.2

**Threat** - Base images downloaded from external container image registries can introduce malware, backdoors, and vulnerabilities.

**Best Practice** - A set of common base container images should be maintained in a container registry that is under the entity’s control.

**Details** -  For this requirement looking at the container images in use in the cluster can be helpful. There will be a mix of internally maintained and externally maintained images, and it's important to understand the process for managing the externally maintained images.

## Section 12.3

**Threat** - The default position of Linux containers, which is to run as root, could increase the risk of a container breakout.

**Best Practice** - Container images should be built to run as a standard (non-root) user.

**Details** -  Generally when enforcing this requirement, an external admission control system (e.g. Kyverno or OPA Gatekeeper) will be in use, so reviewing policies should confirm that this is being enforced. Alternatively reviewing Dockerfiles for the `USER` directive and/or Kubernetes manifests to confirm that the `runAsNonRoot` flag is being set can be helpful.

## Section 12.4

**Threat** - Application secrets (i.e., cloud API credentials) embedded in container images can facilitate unauthorized access.

**Best Practice** - Secrets should never be included in application images. Where secrets are required during the building of an image (for example to provide credentials for accessing source code) this process should leverage container builder techniques to ensure that the secret will not be present in the final image.

**Details** -  Reviewing running containers with secrets scanning software like [Trivy](https://aquasecurity.github.io/trivy/v0.27.1/docs/secret/scanning/) can be helpful here, however it's worth noting that this kind of software can be false-positive heavy. Dockerfiles can also be helpful to note any secrets being added to the image. Another good approach would be to understand the organization's approach to secrets management.

# Conclusion

Container image building is a fundamental part of the container lifecycle, and it's important to understand how to do it securely. This post has looked at some of the PCI requirements that are relevant to this process, and how to review for them.