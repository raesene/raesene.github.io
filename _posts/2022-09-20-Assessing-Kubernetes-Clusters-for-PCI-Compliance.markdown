---
layout: post
title: "The Challenges of Assessing Kubernetes clusters for PCI Compliance" 
date: 2022-09-20 07:45:00 +0100
comments: false
categories: Kubernetes
---

After talking about the release of PCIs recommendations for containers and container orchestration environments, and how it could be applied to Kubernetes clusters in my [last blog](https://raesene.github.io/blog/2022/09/10/PCI-Guidance-for-containers-and-container-orchestration-tools/) I thought that it might be a good idea to discuss some of the general challenges that assessors and auditors might have when looking at Kubernetes environments, as there's quite a few variables that you need to account for. This is part of a longer series on the PCI guidance for containers, and an index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

## What is Kubernetes?

Whilst there is the [open source project](https://github.com/kubernetes/kubernetes) that's not how most companies will deploy Kubernetes, instead they'll make use of a "Kubernetes distribution". To be a certified Kubernetes distribution software has to pass [conformance testing](https://www.cncf.io/certification/software-conformance/) which ensures that it will operate as expected, so there will be a level of common functionality to all Kubernetes distributions. However, there is still a lot of latitude that distribution providers have in terms of the exact configuration and operating model, which an auditor needs to be aware of. At the moment there are 65 different certified distributions 51 different certified "hosted" Kubernetes and 23 different certified installers.

## Managed against Unmanaged

Fundamentally there are two different groups of Kubernetes distributions. A managed distribution occurs where the provider manages the control plane (API Server, Controller manager, scheduler and etcd) and the customer has access to, and manages the workloads that run on the cluster (and possibly manages the worker nodes). Major examples of this kind of approach are Amazon EKS, Azure AKS and Google's GKE.

In a managed installation typically there is no access to look at the configuration of the control plane. This means that from an audit perspective any requirements that relate to precise configuration of the control plane components (for example flags set on the API server) will either be assessed via the provider's control panels/APIs or in some cases they can't be directly assessed at all! For some managed distributions there is a CIS benchmark, but if not an auditor will need to look at each requirement to see what the cloud providers defaults are.

In an unmanaged distribution (e.g. Kubeadm, Rancher), the cluster operator typically has full access to the control plane and can configure any of the components as needed. From an auditing perspective this is easier as everything can be assessed. However there is a general challenge in that things like exact file locations and names are not prescribed by the Kubernetes conformance testing, so you can't assume that things will always be in the same place.

I'll also call out at this point [Red Hat Openshift](https://www.redhat.com/en/technologies/cloud-computing/openshift) as it's kind of a special case. It is a certified Kubernetes but it varies more from core Kubernetes in how it operates, so ideally when assessing it (either in managed or unmanaged form) guidelines being followed should be specifically designed for OpenShift. There is a CIS benchmark for it, so that may be useful.

## Kubernetes Versions

Another variable that needs to be accounted for when assessing a Kubernetes cluster is the version of Kubernetes in use. Kubernetes used to release a new version every 3 months and now does so every 4 months. Whilst only the last 3 versions are in support, many clusters are now running unsupported versions so there is a requirement to know about a variety of versions.

One reason why this is important is that the Kubernetes project will introduce new configuration options and deprecate others from version to version. One good example is the "Insecure API port". This setting went through a [process of deprecation](https://github.com/kubernetes/kubernetes/issues/91506) over a large number of versions. Initially it defaulted to being available over `127.0.0.1:8080` the default then changed to not being on by default, but still available, to not being available but the flag was still present, to not being available at all. Obviously if you use an audit guide for the wrong version of Kubernetes this could lead to a false positive or false negative result.

Another variable that changes over time with Kubernetes is that API versions change. Basically an API can be in "alpha" (not enabled by default), "beta" (which used to be enabled by default but since Kubernetes 1.24 new beta APIs will not be), "GA" always enabled, "deprecated" shouldn't be used but still works or removed where it no longer works. One API which is relevant to auditors, which went through some of this process is PodSecurityPolicy, which was a core feature of Kubernetes security for some versions. Its path was alpha-->beta-->deprecated-->removed, but as beta APIs were generally enabled, it was in use by a large number of clusters.

The overall gist of this section is to emphasise the importance of matching the guidance/audit standard you're using to the version of Kubernetes you're reviewing. Older guides will often lead to incorrect results as defaults change over time. There are some benchmarks (like the CIS benchmarks) which have different benchmarks for different versions of Kubernetes, but in general this will require reviewers to dig into Kubernetes documentation (and sometimes github issues) to find out what the correct settings are.

## Conclusion

The goal of this post was really just to describe some of the complexities that you need to be aware of when assessing or auditing a Kubernetes cluster, it's important not to take a "one size fits all" approach and to tailor the review to the distribution and version that's being assessed. I'd expect that over time more assessment guides will be made available. Also I'd expect we'll see more automated tooling which can help with compliance reviews, however it's important to note that automated tools have to try and deal with all this complexity as well and some of the requirements in the PCI recommendations do not lend themselves to full automation.

Also whilst there are challenges in the precise detail of assessing Kubernetes cluster, that doesn't mean we can't come up with general guidance for the PCI Container Orchestration Recommendations and in the next post(s) I'll take a look at that topic, starting with the [Authentication section](https://raesene.github.io/blog/2022/10/01/PCI-Kubernetes-Section1-Authentication/) 