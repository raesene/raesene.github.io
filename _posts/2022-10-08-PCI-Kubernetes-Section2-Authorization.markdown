---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 2 - Authorization" 
date: 2022-10-08 14:40:00 +0100
comments: false
categories: Kubernetes
---

This is the 2nd part of an in-depth look at how companies running Kubernetes can approach implementing the recommendation of PCI's guidance for container orchestration. The [previous installment](https://raesene.github.io/blog/2022/10/01/PCI-Kubernetes-Section1-Authentication/) looked at authentication, and there's also an [overview post](https://raesene.github.io/blog/2022/09/10/PCI-Guidance-for-containers-and-container-orchestration-tools/) and some notes on the [complexity of assessing security in Kubernetes](https://raesene.github.io/blog/2022/09/20/Assessing-Kubernetes-Clusters-for-PCI-Compliance/) which might be worth reading before getting in to this part. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

# Section 2 - Authorization

Authorization is generally the second step, after authentication, in providing users access to a system's resources and Kubernetes is no different in that regard. Kubernetes supports a number of different types of authorization, and uses a cumulative method to assess a client's rights, so it's important to understand the supported methods in a given cluster and review each of them. 

The Kubernetes documentation has a good list of [authorization modes](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules). In general the most used authorization methods, for user access, are RBAC and Webhook authorization which is sometimes used by managed Kubernetes distributions to integrate with Cloud IAM services. Node authorization is a specialist mode designed to control the rights of kubelet services and ABAC is generally no longer in use.

## Section 2.1

**Threat** - Excessive access rights to the container orchestration API could allow users to modify workloads without authorization.

**Best Practice** - Access granted to orchestration systems for users or services should be on a least privilege basis. Blanket administrative access should not be used.

**Details** - This is a fairly standard "least privilege" style security recommendation but there are a couple of Kubernetes specific cases to consider. Firstly, Kubernetes has a number of built-in clusterroles (`edit`, `view` and `cluster-admin`) which provide a general set of resource access. clusters should not make use of those roles but instead ensure that they review what access users actually require and provide only that access. In particular `cluster-admin` should not be used as this provides completely unrestricted access to the cluster using wildcard (`*`) operators.

## Section 2.2

**Threat** - Excessive access rights to the container orchestration tools may be provided through the use of hard-coded access groups.

**Best Practice** - a. All access granted to the orchestration tool should be capable of modification. b. Access groups should not be hard-coded.

**Details** - With Kubernetes there is one specific instance where a hard-coded admin group is used, which is the use of the `system:masters` group. Any user or service which is a member of this group automatically has cluster-admin access. It's important to note that this access works even if all RBAC rules are removed and any requests from a user in this group are not even sent to authorization webhooks for review, they're just approved at the API server level.

This group was put in place to provide a "break glass" access in the case that a cluster operator had broken the RBAC system, however it is often used by Kubernetes distributions with the first user in the cluster, which can lead to cluster operators continuing to use it for general administration.

Users and services should not be added to this group. If a credential with this access is required, it should be held in a secrets management system and accessed only when required.

## Section 2.3

**Threat** - Accounts may accumulate permissions without documented approvals.

**Best Practice** - Use manual and automated means to regularly audit implemented permissions. 

**Details** - On the face of is, this is a pretty general requirement to review authorization to Kubernetes clusters on a regular basis, however there are some nuances which should be understood when reviewing Kubernetes authorization.

As mentioned earlier there are multiple authorization methods, so if a cluster has RBAC and Webhook authorization activated, the rights contained in both systems need to be reviewed. 

In-built Kubernetes tooling (e.g. `kubectl auth can-i --list`) only takes accounts of the rights supplied via RBAC so it's important to note that other tools are needed for webhook authorization reviews.

Then we come to the sticky problem which we mentioned in the authentication post, that Kubernetes doesn't have a user database. What Kubernetes RBAC does is just take the usernames, group names, requested resource, and requested action and match them against the rules in the RBAC system. It has no idea, for example, how many users are in a group. This makes traditional authorization review techniques a bit tricky.

Essentially the only way to do it reliably is to look at each enabled authentication method, and then go to the repository of user information for that method and get things like group memberships there, an approach which should work for things like OIDC authentication.

A tricky point here is around client certificate authentication as there is generally no record of what the content of an approved client certificate was, after it's been approved, so you're reliant on some form of external record keeping to assess access of client certificates.

Another thing to be aware of when reviewing access to Kubernetes clusters is that there are quite a few resources that can allow for privilege escalation, so they need to be accounted for in the review. There's a Kubernetes documentation page covering [privilege escalation risks](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#privilege-escalation-risks)

Another area to watch for when using automated RBAC review tooling is that Kubernetes doesn't make it easy/possible to enumerate all resource types and operations (this post on [virtual verbs](https://blog.aquasec.com/kubernetes-verbs) in Kubernetes has some details), so care should be taken when relying on them.

# Conclusion

The PCI recommendations for Authorization are a pretty standard set of good practices for multi-user systems, however enforcing them for Kubernetes does require people to take account of some peculiarities in how Kubernetes operates when implementing them. In our next part we'll be l[ooking at section 3 of the guidance on the topic of Workload security](https://raesene.github.io/blog/2022/10/15/PCI-Kubernetes-Section3-workload-security/).