---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 4 - Network Security" 
date: 2022-10-23 11:27:00 +0100
comments: false
categories: Kubernetes
---

This is the fourth part of a series of posts looking at the PCI recommendations for container security as they apply to Kubernetes environments. This time we're looking at Network security. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

Before getting into the details of how the PCI recommendations might apply, it's worth touching briefly on how Kubernetes networking works, at a high level.

Networking is one of the areas where Kubernetes delegates responsibility to third party components via a well defined API, in this case the Container Networking Interface (CNI). This means that there can be a wide variety of implementations, which can make assessing cluster security tricky to talk about generally. At a high level there's two ways that networking is approached in Kubernetes :-

- **Overlay Networking** - Here a separate network space is established for the containers running in the cluster. The CNI component allows all containers to talk to each other in a shared IP address space, regardless of the architecture of the underlying network topology. This has advantages in simplicity of operation. One point to note here is that although typically the overlay network isn't accessible from the underlying network, this isn't a security mechanism as [Kubernetes nodes are routers](https://raesene.github.io/blog/2021/01/03/Kubernetes-is-a-router/) so other hosts on the same subnet as a Kubernetes cluster can get access to containers unless additional restrictions are in place.

- **Direct Networking** - An alternative approach is to provide containers with IP addresses on the same network as the underlying nodes. This is a relatively common implementation from cloud managed Kubernetes distributions such as [AKS](https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md) and [EKS](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html). This removes the complexity of having a separate overlay network, but does mean that things like dealing with possible IP address exhaustion on the network can become a challenge.

In either case the default for Kubernetes is to allow unrestricted access to and from containers running in the cluster, so there is a requirement to add restrictions to lock this down from its initial open state.

# Section 4.1

**Threat** - Container technologies with container networks that do not support network segmentation or restriction allow unauthorized network access between containers.

**Best Practice** - Container orchestration tool networks should be configured on a default deny basis, with access explicitly required only for the operation of the applications being allowed.

**Details** - There's a couple of pieces that need to be checked and configured in Kubernetes to address this recommendation. The first is ensuring that the Container Networking solution (CNI) in place supports Kubernetes network policies which are used to restrict network traffic. Whilst most do there are some, like Flannel, which do not.

Assuming that the CNI is in place, the next step is to ensure that a default deny policy is in place for each Kubernetes namespace. The way that network policies work, there needs to be an *ingress* and *egress* policy per namespace applying that policy and the policies must apply to every workload. In general network policies are applied using workload selectors and if there are no policies that apply to a given workload, all traffic is allowed in that direction, so it's important to ensure that the base deny policy is in place. There's a good example of this kind of policy [here](https://github.com/ahmetb/kubernetes-network-policy-recipes/blob/master/03-deny-all-non-whitelisted-traffic-in-the-namespace.md) (as well as a lot of other good network policy examples).

Another point to mention here is that some CNIs (e.g. Calico or Cilium) have extended the network policy model and have their own network policy style objects which can apply default policies more easily. When reviewing a cluster, it is important to first check the CNI in use and then check both base network policies *and* the additional network policy types, if they're supported.

# Section 4.2

**Threat** - Access from the container or other networks to the orchestration component and administrative APIs could allow privilege escalation attacks.

**Best Practice** - Access to orchestration system components and other administrative APIs should be restricted using an explicit allow-list of IP addresses.

**Details** - This is another area where there aren't any policies implemented by default so it requires configuration on a per cluster basis. From within the cluster this would likely be handled by the first section, so in terms of additional requirements we're looking at access to orchestration APIs from external sources. For unmanaged Kubernetes this needs to be handled by adding network firewall restrictions to the IP addresses of the control plane and worker nodes in the cluster. For managed Kubernetes, generally access to the Kubernetes API server is handled at a cloud configuration layer. It's worth noting that the "big 3" managed cloud distributions all default to placing the Kubernetes API server directly on the Internet with no IP address restrictions, however they do support restricting access to whitelisted ranges or making it only available from cloud internal IP address ranges.

# Section 4.3

**Threat** - Unencrypted traffic with management APIs is allowed as a default setting, allowing packet sniffing or spoofing attacks.

**Best Practice** - All traffic with orchestration system components APIs should be over encrypted connections, ensuring encryption key rotation meets PCI key and secret requirements.

**Details** - From a core Kubernetes perspective, the default management API access should be encrypted, however there are a couple of legacy Kubernetes management APIs which allow for unencrypted access, and care should be taken to ensure that these are not activated or used. Firstly the Kubernetes insecure API operates over an unencrypted connection to port 8080/TCP by default. This API should never be enabled as, in addition to not requiring encryption, it does not require authentication!  The other core Kubernetes API which may be available without encryption is the read-only Kubelet service which defaults to being available on port 10255/TCP on any nodes that have the Kubelet running. Whilst this service is read-only it does include information which may be sensitive and like the insecure API service, it does not have authentication support, so should not be used in production clusters.

In addition to core Kubernetes APIs, it's worth noting that supporting systems (e.g. logging and monitoring) should require encryption for access where they are communicating across any network, including the container network.

# Conclusion

Network security is another core part of creating a secure Kubernetes environment and, like workload security which we covered last time, Kubernetes default position is not to have any restrictions in place, so it falls to cluster operators to ensure that Kubernetes environments are appropriately locked down. Next time we'll be looking at the [PKI Recommendations](https://raesene.github.io/blog/2022/10/29/PCI-Kubernetes-Section5-PKI/)

