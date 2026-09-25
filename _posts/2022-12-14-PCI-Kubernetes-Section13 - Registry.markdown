---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 13 - Registry" 
date: 2022-12-14 15:27:00 +0100
comments: false
categories: Kubernetes
---

This is the thirteenth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at the "Registry" section which talks about Container Registry controls. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

Container registries are a key part of an Kubernetes environment as they store the images which are used to create the containers that run the applications hosted in the cluster. Also as I mentioned in the post on [patching](https://raesene.github.io/blog/2022/12/03/PCI-Kubernetes-Section10-Patching/), software updates should be carried by re-building images, pushing them to the registry and then rolling out the new version of the application. This means that the registry is a key part of the update process.

From a technology standpoint container registries are relatively simple. They provide an HTTP API which follows the [OCI distribution specification](https://github.com/opencontainers/distribution-spec). The OCI distribution specification is a standard for how to interact with a registry, it doesn't specify how the registry is implemented. There are a number of different implementations of the specification, including [Docker Hub](https://hub.docker.com/), [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/), [Google Container Registry](https://cloud.google.com/container-registry) and [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/). The specification is also implemented by [Harbor](https://goharbor.io/), an open source registry implementation.

OCI Registries can also be used to store other types of artifacts, such as Helm charts, using projects like [ORAS](https://oras.land/) but for the purposes of this post I'm going to focus on the storage of container images.

An important point about registries is that public images in registries like Docker Hub are not necessarily maintained/curated. There is a set of [Docker Official Images](https://docs.docker.com/docker-hub/official_images/) which are generally maintained (although some are [deprecated](https://blog.aquasec.com/docker-official-images) so care is still needed) but there are also a lot of images which are not maintained by anyone. This means that you should be careful about using images from public registries, especially if they are not maintained by a trusted source. You should also be careful about using images from private registries that you don't control. If you're using a private registry you should be careful about who has access to it and what images are stored in it.

In terms of managing images for production systems, the safest approach is to combine internally managed images hosted in a private registry with public images where required. The public images should be signed using something like [cosign](https://github.com/sigstore/cosign) or pinned to specific SHA256 hashes. This means that you can be sure that the image you're using is the one you expect and that it hasn't been modified.

In terms of the PCI requirements there's four in this section.

## Section 13.1

**Threat** - Unauthorized modification of an organization’s container images could allow an attacker to place malicious software into the production container environment.

**Best Practice** - a. Access to container registries managed by the organization should be controlled. b. Rights to modify or replace images should be limited to authorized individuals.  

**Details** - Exact details of how this is achieved will depend on the registry or registries in use, but access control should be applied to any modification of the images in the registry. Reviewing for this could be done by listing the container images in use in a cluster and then testing to see whether these images are accessible publicly/without authentication and confirming that attempts to modify them without authentication are rejected.

## Section 13.2

**Threat** - A lack of segregation between production and non-production container registries may result in insecure images deployed to the production environment.

**Best Practice** - Consider using two registries, one for production or business-critical workloads and one for development/test purposes, to assist in preventing image sprawl and the opportunity for an unmaintained or vulnerable image being accidentally pulled into a production cluster.

**Details** - Reviewing this recommendation would likely largely be done by speaking to the people responsible for the PCI environment and confirming which registries are used. A check coule be carried out again by listing the images in use in the cluster and then checking to see whether they are from a production or non-production registry.

## Section 13.3

**Threat** - Vulnerabilities can be present in base images, regardless of the source of the images, via misconfiguration and other methods.

**Best Practice** - If available, registries should regularly scan images and prevent vulnerable images from being deployed to container runtime environments.

**Details** - Vulnerability scanning at the registry level is one of the better approaches to container vulnerability scanning as the registry should be the canonical source of images. Some registries will provide image scanning functionality directly, or third-party scanning tools such as [Trivy](https://github.com/aquasecurity/trivy) can be used to scan images.

## Section 13.4

**Threat** - Known good images can be maliciously or inadvertently substituted or modified and deployed to container runtime environments..

**Best Practice** - Registries should be configured to integrate with the image build processes such that only signed images from authorized build
pipelines are available for deployment to container runtime environments.

**Details** - Image signing is a good control to help ensure that the images in the registry are the ones that you expect. The main ecosystem currently in use for this is [sigstore](https://www.sigstore.dev/) which allows for signatures to be stored on a transparency log. This allows for the signatures to be verified without having to trust the registry.

# Conclusion

Container registries are a vital part of any Kubernetes clusters, so it's not surprising that there are control recommendations relating to them. Used correctly, they can help ensure that Kubernetes clusters are running trusted and up to date images.