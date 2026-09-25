---
layout: post
title: "PCI Guidance for Containers and Container Orchestration Tools" 
date: 2022-09-10 08:45:00 +0100
comments: false
categories: Kubernetes
---

Yesterday, the [PCI Council issued a new information supplement](https://blog.pcisecuritystandards.org/new-information-supplement-guidance-for-containers-and-container-orchestration-tools?utm_campaign=Stakeholder%20Engagement&utm_content=220770109&utm_medium=social&utm_source=twitter&hss_channel=tw-20256309) that should be of specific interest to anyone using container technologies like Docker and podman and Container orchestration technologies like Kubernetes and OpenShift to process cardholder transactions. 

[The Guidance](https://docs-prv.pcisecuritystandards.org/Guidance%20Document/Containers%20and%20Container%20Orchestration%20Tools/Guidance-for-Containers-and-Container-Ochestration-Tools-v1_0.pdf) has been in development since last year and is intended to provide more specific information about how PCI in-scope entities should secure containerized environments.

Whilst the whole document has a lot of information and is well worth a read, probably the most interesting section to a lot of companies is likely to be the threat and best practice table (3.1). This table has 16 sections of recommendations for the secure operation of containers and container orchestration tools.

The guidance provided by the PCI council is intentionally vendor agnostic but it's possible to derive how it might apply to the most common container orchestration environment, Kubernetes.

This is a short list of recommendations but there is most detail in the series of posts that I've written on this topic. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

## 1. Authentication

There's a couple of recommendations in this section which require specific attention. In section 1.2 the guidance recommends avoiding the use of generic administration accounts, and in Kubernetes this would apply to the first account commonly created with clusters (kubernetes-admin) as well as any other generic account.

Additionally in section 1.3 there is a recommendation to ensure that all credentials used can be revoked. This is particularly significant to Kubernetes as it means that client certificate authentication and the Kubernetes TokenRequest API should be used with extreme care as neither have a revocation system in place (without either re-issuing all certificates for the cluster in the case of client certificate authentication, or deleting the serviceaccount/pod/secret tied to the token in the case of the TokenRequest API)

Also, section 1.5 will apply to the automatic provisioning of service accounts to Kubernetes workloads, which is a common default. The recommendation here is to only supply API server credentials to workloads where explicitly required.

Lastly in Section 1.6 a recommendation for administrative users to use MFA to authenticate to cluster APIs. Again, this would require the use of an external authentication system as none of the in-built Kubernetes authentication mechanisms support MFA.

## 2. Authorization

In addition to the generic recommendation use least privileged access in section 2.1, which would include avoiding the use of the `cluster-admin` clusterrole in Kubernetes, section 2.2 explicitly recommends against using "hard coded access groups". The best known of these in Kubernetes is the `system:masters` group, which should not be used for general administration as it is only required in "break glass" scenarios where the RBAC system is unavailable.

## 3. Workload security

There are a couple of quite important recommendations in this section. 3.1 requires that workloads in the cluster are prevented from accessing the underlying cluster node resources. In Kubernetes that requires either the use of [pod security admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) or a 3rd party admission controller like [Kyverno](https://kyverno.io/) or [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper).

Next section 3.2 recommends that containers run in the cluster target known versions. Ideally this should be done by checking cryptographic signatures on images (for example with [cosign](https://github.com/sigstore/cosign)) or failing that SHA-digests. The reason for this being that tags on container images are generally mutable so don't reliably point to the same content.

There's also a general recommendation in section 3.3 to source images for production systems from an in-house controlled container registry.

## 4. Network Security

Possibly the most important recommendation here is, in section 3.1, to use a default deny network policy in Kubernetes clusters. this can be done by ensuring that a default policy is set on every namespace addressing both ingress and egress traffic, with specific requirements being specified to white-list access on top of that.

There's also a recommendation in section 4.2 to avoid having administrative APIs (like the Kubernetes API server) exposed to the general Internet. This is [something I've written about before](https://raesene.github.io/blog/2022/07/03/lets-talk-about-kubernetes-on-the-internet/) as there are a lot of clusters visible on the Internet.

## 5. PKI

In section 5.1 there's another reminder to avoid using certificate-based authentication if revocation is not supported, which is the case in Kubernetes. Whilst this form of authentication is required for system-component --> system-component authentication it should not be used for user or service based authentication.

## 6. Secrets Management

In section 6.1 the guidance recommends using a dedicate secrets management tool for storing sensitive information. In Kubernetes one pattern I've seen that should definitely be avoided here is using config maps to store secret information as they would not comply with this recommendation.

## 7. Container Orchestration Tool Auditing

Section 7.1 recommends not only that auditing is enabled (which is often not the default with Kubernetes distributions) but that audit logs are stored and processed on a centralized system to avoid the risk of tampering.

## 8. Container Monitoring

Section 8.1 applies similar guidance to the previous section recommending that container logs are stored centrally to ensure their availability in the case of a security incident.

Section 8.2 recommends that container security monitoring tools that can detect unauthorized modification of container files (e.g. [Tracee](https://www.aquasec.com/products/tracee/), [Falco](https://falco.org/) or [Tetragon](https://github.com/cilium/tetragon))

## 9. Container Runtime Security

Section 9.1 recommends that for high-risk workloads hypervisor (e.g. Firecracker) or sandbox (e.g. gVisor) style isolation is considered, due to the larger attack surface of standard Linux container isolation.

Section 9.2 will be specifically relevant to anyone using Windows containers in PCI environments. As I talked about [last week](https://raesene.github.io/blog/2022/09/03/Fun-With-Windows-Containers-Popping-Calc/) Windows process isolated containers are not considered by Microsoft to provide a security barrier, so Hyper-V based containers should be used to provide appropriate isolation. A complicating factor here is that currently Hyper-V based Windows containers are not supported under Kubernetes.

## 10. Patching

Section 10.1 makes the recommendation that security patches are applied to all container orchestration tools. One challenge in Kubernetes environments is staying within the support window of the product (currently 12 months) so attention should be paid to this.

Section 10.2 recommends that cluster nodes are regularly upgraded. Of specific importance here is Linux kernel upgrades where nodes will likely need rebooted for them to take effect.

Section 10.3 recommends that container images are regularly scanned updated and the containers redeployed so that those updates are effective in the cluster.

## 11. Resource Management

Section 11.1 recommends that all workloads in a cluster have resource limits applied. With containers sharing resources on cluster nodes, it is generally important to try and reduce the risk of noisy neighbours.

## 12. Container images building

The recommendations in this section aren't really specific to container orchestration, they apply wherever containers are used in in-scope environments.

A recommendation to use minimal base images in a registry under the organization's control are covered in 12.1 and 12.2, while a standard piece of container security advice to run containers as non-root users is mentioned in section 12.3.

## 13. Registry

The recommendations in this section relate to the security of container image registries. Section 13.3 recommends integrating container vulnerability scanning into the registry to ensure that images are regularly scanned and updated.

## 14. Version Management

This section covers some best practices related to ensuring that version management is used for cluster resource.

## 15. Configuration Management

This section deals with ensuring that testing and configuration standards are in place for all container/container orchestration components.

## 16. Segmentation

This section relates to segmentation of clusters. Recommendation 16.1 covers the idea that dedicated clusters should be used for high-security components with 16.2 covering the idea of using dedicated node pools where that's not possible. In general, effectively creating *security* segmentation on Kubernetes is a difficult task requiring isolation of workloads, the network and Kubernetes API services. Generallym the surest security barrier is placed at the cluster level.

## Conclusion

It's good to see additional guidance for container orchestration environments being developed as these technologies are more widely used in regulated environments. with this guidance there are definitely some challenging areas for Kubernetes based environments where defaults will need to be adapted to include PCI's recommendations. Next time we'll take a look at the [challenges in assessing the security of Kubernetes clusters for PCI compliance](https://raesene.github.io/blog/2022/09/20/Assessing-Kubernetes-Clusters-for-PCI-Compliance/)