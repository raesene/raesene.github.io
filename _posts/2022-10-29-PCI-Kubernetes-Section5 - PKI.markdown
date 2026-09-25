---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 5 - PKI" 
date: 2022-10-29 11:27:00 +0100
comments: false
categories: Kubernetes
---

This is the fifth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at PKI. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

This section gets into an area that's fairly well trodden for financial services companies which is PKI management. Organizations that are used to more traditional approaches to PKIs might be somewhat surprised by how Kubernetes makes use of Public Key Infrastructure.

In a "vanilla" Kubernetes cluster, there will generally by three entirely separate Certificate authorities each with their own keys (details [here](https://kubernetes.io/docs/setup/best-practices/certificates/)). The first one is the main CA for the cluster, this is used for cases where components or users want to authenticate to the Kubernetes API server. certificates signed by this CA are trusted by the API server and can be used by other components as well.

The second CA is used for authentication between the API server and etcd. This is needed as the way that etcd runs in Kubernetes clusters, it provides full access to any client certificate issued by a specified CA, so if it used the main CA any user with access to a client certificate would be able to extract all the contents of the etcd database (leading to a privilege escalation issue).

The third one is used where there are "extension API servers" in the cluster, but typically the keys are created even if that feature is not used. 

From a security perspective the first thing to note is that the certificate authority private keys will be held in the clear on control plane node disks, so controlling access to those directories (and auditing access to them) is important. When looking at managed Kubernetes where typically the cluster operator doesn't have access to the control plane nodes, this isn't as much of an issue, although it is still possible to issue new client certificate from the "main" CA using the CertificateSigningRequest API.

Another security consideration, as we've mentioned [before](https://raesene.github.io/blog/2022/10/01/PCI-Kubernetes-Section1-Authentication/) when covering authentication, Kubernetes [doesn't support certificate revocation](https://github.com/kubernetes/kubernetes/issues/18982), so it can be difficult to manage risks around compromised private keys.

## Section 5.1

**Threat** - Inability of some container orchestration tool products to support revocation of certificates may lead to misuse of a stolen or lost certificate by attackers

**Best Practice** - a. Where revocation of certificates is not supported, certificate-based authentication should not be used. b. Rotate certificates as required by PCI or customer policies or if any containers are compromised.

**Details** - This point essentially duplicates section 1.3 in the authentication section of the guidance. To re-state what was said there essentially organizations should avoid using Kubernetes client certificate authentication wherever possible. Where the cluster uses client certificate authentication for the initial user provided with the cluster, that credential should be stored safely offline and not used for general cluster administration duties.

## Section 5.2

**Threat** - PKI and Certificate Authority services integrated within container orchestration tools may not provide sufficient security outside of the container orchestration tool environment, which could lead to exploitation of other services that attempt to use this chain of trust.

**Best Practice** - The certificates issued by orchestration tools should not be trusted outside of the container orchestrator environment, as the container orchestrator’s Certificate Authority private key can have weaker protection than other enterprise PKI trust chains.

**Details** - This ties to the points mentioned at the top of this blog around how Kubernetes does PKI key management. Having on-line private keys without passphrase protection means that any attacker who is able to get even temporary read access to the CA key will be able to persistently be able to create certificates for that CA. Where an organisation using Kubernetes needs to use a PKI they should use an externally managed one, for example making use of cloud service provider key management services.

# Conclusion

PKIs are essentially intertwined with Kubernetes architecture, but in general for production PCI clusters it's likely the companies should only use Kubernetes PKI options where absolutely required and instead use externally managed PKI services for any other requirements they may have. Next time we'll be looking at the [Secrets Management Recommendations](https://raesene.github.io/blog/2022/11/06/PCI-Kubernetes-Section6-Secrets-Management/)
