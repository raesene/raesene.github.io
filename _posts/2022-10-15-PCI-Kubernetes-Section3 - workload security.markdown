---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 3 - Workload Security" 
date: 2022-10-15 09:00:00 +0100
comments: false
categories: Kubernetes
---

This is the 3rd part of an in-depth look at how companies running Kubernetes can approach implementing the recommendation of PCI's guidance for container orchestration. The [previous installment](https://raesene.github.io/blog/2022/10/08/PCI-Kubernetes-Section2-Authorization/) looked at authorization, and there's also an [overview post](https://raesene.github.io/blog/2022/09/10/PCI-Guidance-for-containers-and-container-orchestration-tools/) and some notes on the [complexity of assessing security in Kubernetes](https://raesene.github.io/blog/2022/09/20/Assessing-Kubernetes-Clusters-for-PCI-Compliance/) which might be worth reading before getting in to this part. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

# Section 3 - Workload Security

Running containers is obviously the main thing that most Kubernetes clusters are responsible for, so it makes sense that there's a section of the guidance dedicated to them. Before we talk about the specific recommendations, it's important to cover off a couple of base concepts.

Containers are just Linux (or Windows) processes. When run under Kubernetes the defaults for both is to use operating system features to isolate those processes from each other and the underlying host. So you can think of Kubernetes as essentially distributed remote command execution :)

By default, with no additional controls, any user who can launch containers into a cluster can get `root` access to the underlying cluster node simply (I've covered how to do that before with [the most pointless kubernetes command ever](https://raesene.github.io/blog/2019/04/01/The-most-pointless-kubernetes-command-ever/) which is based on [the most pointless docker command ever](https://zwischenzugs.com/2015/06/24/the-most-pointless-docker-command-ever/) from Ian Miell)).

Docker, and by extension Kubernetes, have a flexible security model where individual restrictions can be removed or enhanced. The defaults were generally chosen for ease of operation, so it's not surprising that they need specific hardening recommendations for production PCI environments.

## Section 3.1

**Threat** - Access to shared resources on the underlying host permits container breakouts to occur, compromising the security of shared resources.

**Best Practice** - Workloads running in the orchestration system should be configured to prevent access to the underlying cluster nodes by default. Where granted, any access to resources provided by the nodes should be provided on a least privilege basis, and the use of “privileged” mode containers should be specifically avoided.

**Details** - So as we've mentioned Kubernetes requires specific additional controls to be put in place to stop containers getting access to underlying cluster resources. The mention of `privileged` in the recommendation refers to the Docker `--privileged` flag which essentially just removes all the security isolation between a container and the underlying node. It's sometimes used as a shortcut to avoid having to work out exactly what access a container needs to the underlying node.

In terms of how these restrictions are put in place, the picture can be a bit complex. In older versions of Kubernetes a feature called Pod Security Policy was available which could be used to restrict workloads. However this was removed [in the latest version of Kubernetes](https://kubernetes.io/blog/2022/08/23/podsecuritypolicy-the-historical-context/). 

There is a replacement feature within Kubernetes, which can be used to implement restrictions on Workloads called [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/), however this may not be suitably flexible for all companies needs, so many organizations make use of external admission control software to place restrictions on workloads running in the cluster. In the open source world, prominent options for this include [Kyverno](https://kyverno.io/), [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper), [jsPolicy](https://www.jspolicy.com/) and [Kubewarden](https://www.kubewarden.io/). Also a special note for OpenShift here which has it's own mechanism [Security Context Constraints](https://docs.openshift.com/container-platform/4.11/authentication/managing-security-context-constraints.html).

Having covered how restrictions on workloads would be put in place, we also need to think about what restrictions to put in place. At this point it's important to note that some system workloads do need access to the underlying cluster nodes to operate, so for example the `kube-proxy` component needs to modify networking components so will need access to that.

For general workloads the goal is to avoid giving them rights that would allow for access to the underlying host. The Kubernetes project has created [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) to document which settings are needed (as there are quite a few). Enforcing at least the **baseline** policy and ideally using the **restricted** policy for all general workloads should prevent the processes running in containers from accessing the underlying host.

So if you're reviewing a cluster for PCI there's a couple of actions 

* Ensure that one of the systems that can be used to restrict workloads is in place and operational

* Review the policies applied to the workloads in the cluster to assess how well they meet the requirements of Pod Security Standards.

## Section 3.2

**Threat** - The use of non-specific versions of container images could facilitate a supply chain attack where a malicious version of the image is pushed to a registry by an attacker.

**Best Practice** - Workload definitions/manifests should target specific known versions of any container images. This should be done via a reliable mechanism checking the cryptographic signatures of images. If signatures are not available, message-digests should be used.

**Details** - Container images are the standard way of packaging the software that will run on your Kubernetes clusters, and they're typically pulled from container registries, either public ones like Docker Hub or private ones under the organization's control. Obviously it's important to ensure (as much as possible) that you know what's inside the container image before you run it. Within registries versions of images are typically denoted based on "tags" and if you don't specify a tag you get whatever the latest version of that image is, which is clearly not great in terms of knowing what you're running.

Also, depending on the registry, it may be possible to change what image a tag points to, so again it's not ideal to rely solely on image tags, although if an internal registry is used it might be possible to establish trust based on how that registry and its tags are managed. Without additional software, one option is to use images based on a specific SHA-256 hash, a mechanism which is generally supported by container software, although it's important to note that this is quite a cumbersome thing to do as it means you need to change the hash *every* time the image is patched or changed in any way.

It's also possible to use digital signing to improve the trust in the workloads you run, but this does require additional software. Specifically it need a signing tool to sign the images and also software to validate the signatures when the container images are deployed to the Kubernetes cluster. For the first part the most common tool is [cosign](https://github.com/sigstore/cosign), and it's possible to validate cosign signed images using the admission control software we mentioned in the previous section.

* Ensure that all images running in the cluster use either a tag (supported by processes to ensure tag integrity), SHA-256 hash or digital signatures to validate their integrity.

* Review how these mechanisms are enforced, reviewing admission controller policies that are in use on the cluster.

## Section 3.3

**Threat** - Containers retrieved from untrusted sources may contain malware or exploitable vulnerabilities

**Best Practice** - All container images running in the cluster should come from trusted sources. 

**Details** - This requirement goes alongside the previous one, in making the point that container image assurance is a key concern for Kubernetes. It's important to note that in the vast majority of cases, Container registries do not curate their images so there is a risk of supply chain attacks at that level.

The safest option is to ensure that all images running in the cluster are sourced from a container registry that is under the control of the organization, and that security checks are carried out whenever new images are added to this registry. This does add overhead to managing the cluster as 3rd party software (e.g. helm charts) will generally assume that it can pull images from whichever registry the software vendor uses.

Where images need to be sourced from external registries mechanisms like using specific SHA-256 hashes or signed images can help to provide assurances that the images can be trusted (assuming of course you trust the project/vendor that created those images).

* All images should come from either an internally controlled registry or where coming from external sources be validated before deployment.

# Conclusion

Workload security is a fundamental part of any container security architecture. PCI's requirements are (as with previous parts) fairly general good practices, however implementing them in a Kubernetes environment could require considerable effort. Planning out how to comply with these requirements is best done during a planning phase, to reduce the potential for impact to running workloads. Next time we'll be looking at the [Network Security Recommendations](https://raesene.github.io/blog/2022/10/23/PCI-Kubernetes-Section4-network-security/)