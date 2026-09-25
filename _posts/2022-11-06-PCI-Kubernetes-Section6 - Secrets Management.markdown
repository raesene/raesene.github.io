---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 6 - Secrets Management" 
date: 2022-11-06 11:27:00 +0100
comments: false
categories: Kubernetes
---

This is the sixth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at secrets management. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

Secrets management is an important part of any containerized environment for a couple of reasons. The main one is that when using containers for deploying your workloads you have a dilemma about where to put any secrets that the container might need during its operation (e.g. credentials for a backend database). Secrets cannot be effectively stored with the images used to create them as those images are often stored on public or semi-public registries and also it would mean you needed to rebuild the image every time you changed a credential, which is unlikely to be practical at scale.

Also containers are ephemeral so manually adding secrets to them at runtime won't be effective, as the container may be rescheduled to another server at any time, where it would be re-created from the image. So we need some automated way of injecting any credentials required by the container when it starts on a given Kubernetes cluster node.

The general answer to that is some form of secrets management system, which can handle providing the right secrets to the right containers, when they start-up. Kubernetes has an in-built [secrets](https://kubernetes.io/docs/concepts/configuration/secret/) facility, which is one option. These are just items stored in the Kubernetes datastore and provided as configured to containers. Some people using Kubernetes have been known to use ConfigMaps for storing secrets, but this isn't a great idea (for reasons I detailed [here](https://blog.aquasec.com/kubernetes-configmap-secrets)).

A point that regularly gets raised about Kubernetes secrets is that, by default, they're not encrypted on disk. Kubernetes provides an option to ensure that this happens, but it needs to be configured. For managed Kubernetes services (e.g. EKS, GKE, AKS) the etcd database is stored by the cloud provider so it's arguable about how much difference it makes encrypting on disk, but generally the option is still available.

In addition to Kubernetes in-built features, there are a wide range of external dedicated secrets management services which will work with Kubernetes. Either cloud provider based options or software like [Hashicorp Vault](https://www.vaultproject.io/) can be used.

So with a general background on this topic, what does PCI specifically recommend.


## Section 6.1

**Threat** - Inappropriately stored secrets, including credentials, provided through the container orchestration tool, could be leaked to  unauthorized users or attackers with some level of access to the environment.

**Best Practice** - All secrets needed for the operation of applications hosted on the orchestration platform should be held in encrypted
dedicated secrets management systems.

**Details** - The intent of this best practice is really around the points discussed above, which is that storing secrets in images, or using general configuration management systems is not appropriate and should be avoided. Also the requirement for encryption means that , if using Kubernetes secrets, you need to enable the option that makes them encrypted on-disk.

## Section 6.2

**Threat** - Secrets stored without version control could lead to an outage if a compromise occurs and there is a requirement to rotate them quickly.

**Best Practice** - Apply version control for secrets, so it is easy to refresh or revoke it in case of a compromise.

**Details** - This is really an availability concern, rather than one around the confidentiality of the secrets. Applying it with base Kubernetes secrets would likely require manual work to define versions of secrets. 

# Conclusion

Secrets management is an inevitable part of container based deployments, and something which needs to be carefully considered when moving to a system like Kubernetes. The PCI recommendations are as with most of them, fairly general good practice, but there are a couple of specifics which apply in Kubernetes to be considered.
