---
layout: post
title: "PCI Compliance for Kubernetes in detail - Part 1 - Authentication" 
date: 2022-10-01 14:45:00 +0100
comments: false
categories: Kubernetes
---

Having taken a high-level look at how the [PCI guidance for container orchestration could apply to Kubernetes environments](https://raesene.github.io/blog/2022/09/10/PCI-Guidance-for-containers-and-container-orchestration-tools/), and some of the [challenges in auditing/assessing Kubernetes environments](https://raesene.github.io/blog/2022/09/20/Assessing-Kubernetes-Clusters-for-PCI-Compliance/), I thought it would make sense to start getting into the details of the recommendations and see how in-scope organizations could look at meeting their requirements when using Kubernetes. Whilst this post is structured round the PCI recommendations, it would hopefully be helpful in general for Kubernetes security. An index of the posts in this series can be found [here](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html).

# Section 1 - Authentication

The first section of the risks and good practices table starts with Authentication. Obviously this is a key security control in most environments and something which companies need to consider. It's also a slightly tricky topic in Kubernetes, as the authentication options provided by the base open source project aren't generally considered suitable for production use, leaving distribution makers and cluster operators with the task of ensuring that secure authentication is in place on their clusters. Some of the PCI recommendations do reflect this challenge.

## Section 1.1

**Threat** - Unauthenticated access to APIs is provided by the container orchestration tool, allowing unauthorized modification of workloads.

**Best Practice** - All access to orchestration tools components and supporting services for example, monitoring from users or other services should be configured to require authentication and individual accountability.

**Details** - Requiring authentication for API access is a pretty obvious first control and there's a couple of ways in which this requirement applies to Kubernetes, the APIs provided by Kubernetes itself and then supporting service APIs.

### Kubernetes APIs

Kubernetes runs a number of services which are exposed to the network. In general these require authentication for any sensitive operations although there is often some level of anonymous access required for some paths and when hardening or auditing a cluster, removing that access might be considered.

An important point when reviewing or securing these APIs is that in managed Kubernetes distributions (e.g. EKS, GKE, AKS) it is not possible for cluster operators to directly change the configuration of most of the APIs unless the cloud provider makes that available. The exception is the Kubelet which runs on worker nodes which are available to the cluster operator (unless it uses a "serverless" model like EKS Fargate)

#### Kubernetes API Server

This listens on a variety of ports depending on the distribution in use. Common options are 443/TCP, 6443/TCP and 8443/TCP. In most distributions the `anonymous-auth` flag will be set to `true`. This provides access to unauthenticated users to specific paths specified in the RBAC configuration of the cluster. For example in a Kubeadm cluster the following paths are available without authentication

```yaml
rules:
- nonResourceURLs:
  - /healthz
  - /livez
  - /readyz
  - /version
  - /version/
  verbs:
  - get
```

These are generally for liveness checks, but the `/version` endpoint does provide information useful to attackers like precise version information.

In terms of compliance recommendations for the API server the main one is

* Disable anonymous authentication where possible, where it is required, ensure that minimal paths are available to unauthenticated users.

#### Controller Manager

Access to the controller manager is generally allowed over port 10257/TCP (can vary with version and distribution). In terms of anonymous access a small number of paths can be specified as a command line flag to allow access, the default settings is as below

```
--authorization-always-allow-paths strings     Default: "/healthz,/readyz,/livez"
```

In terms of recommendations :-

* Review paths which are allowed for anonymous access to ensure that no sensitive data is accessible without authentication.

#### Scheduler

Access to the scheduler is generally allowed over port 10259/TCP (can vary with version and distribution). In terms of access this is very similar to the controller manager, the same parameter and default exists, and the recommendation would be the same. In general for both these services there aren't a lot of good reasons for direct access so outside of health checking there shouldn't be much of a requirement for unauthenticated access.

#### Kubelet

The Kubelet runs on every worker node (and possibly control plane nodes). Access is via 10250/TCP. The kubelet's configuration with regards to anonymous access is a bit odd and not the same as either the scheduler or controller manager. anonymous access defaults to being allowed, so it's a requirement of the distribution that they disable it (either on the command line or in the kubelet's configuration file). 

Requests to the root path of the server will return 404, but requests to meaningful paths (like `/pods/`) will return 401 (if anonymous authentication is disabled) or 403 (if anonymous authentication is enabled).

* Ensure that Kubelet anonymous authentication is disabled unless explicitly required for the operation of the cluster.

#### Etcd

Etcd, whilst not specifically part of the Kubernetes project, is a core part of most Kubernetes distributions. Generally it listens on ports 2379/TCP and 2380/TCP. In most Kubernetes distributions there's no anonymous access to it by default, client certificate authentication is used, so the recommendations are quite simple

* Ensure that etcd is configured to require authentication for all requests.

### Supporting Services

The guidance also references supporting services. this is a bit of a general term and in Kubernetes cluster's you'll find a lot of supporting services for things like logging, monitoring, application lifecycle management and others. There have been a bit of a history of services not requiring authentication by default, and even in some cases not providing the option of authentication, so it's an important point to consider.

So the recommendation here is a bit generic, and will require cluster operators (and auditors) to do some investigation of clusters they're securing or reviewing.

* Review all services deployed to the cluster and ensure that they are not available without authentication. Specifically the services should not rely on the container network as being "trusted" and should still require authentication for requests from any location.

## Section 1.2

**Threat** - Generic administrator accounts are in place for container orchestration tool management. The use of these accounts would prevent the non-repudiation of individuals with administrator account access.

**Best Practice** - All user credentials used to authenticate to the orchestration should be tied to specific individuals. Generic credentials should not be used. When a default account is present and cannot be deleted, changing the default password to a strong unique password and then disabling the account will prevent a malicious individual from re-enabling the account and gaining access with the default password.

**Details** - In terms of managing this recommendation for Kubernetes there's one key area to consider. Most Kubernetes distributions and services will provide an initial user which is created as part of the cluster setup. This user generally has full access to the cluster via the [system:masters](https://blog.aquasec.com/kubernetes-authorization) group and a generic name (for example `kubernetes-admin`). This account should not be used for general administration as obviously there's no way to audit access using it (and also its rights cannot be easily revoked).

* Ensure that if a default administrator account is provided by the Kubernetes distribution or service, this account is not used for general administrative purposes. Instead it should be held in an appropriate secrets management system and used for "break glass" purposes only.

## Section 1.3 

**Threat** - Credentials, such as client certificates, do not provide for revocation. Lost credentials present a risk of unauthorized access to cluster APIs.

**Best Practice** -  All credentials used by the orchestration system should be revokable.

**Details** - This requirement is a fairly obvious one when considering authentication to secure systems. We want to have the option to revoke any credentials that are present for the cluster in case a user has their credentials compromised and also to ensure that our joiners/movers/leavers processes are able to ensure that users only have access to systems that is required by their role.

This requirement is particularly important when the system in question is Internet facing as [many Kubernetes clusters](https://raesene.github.io/blog/2022/07/03/lets-talk-about-kubernetes-on-the-internet/) are.

Where this is somewhat complex in Kubernetes is that the most commonly used forms of authentication available in the base open source project do not allow for revocation. 

### Client Certificate Authentication

One of the main authentication methods available in Kubernetes is client certificate authentication. It's used by internal components for authentication (e.g. the kubelet uses a client certificate to communicate with the Kubernetes API server), often the default first user account provided on cluster setup will be a client certificate, and there is an API provided by the Kubernetes API server to create new client certificates for authentication.

From a PCI compliance standpoint the challenge is that there is [no support for client revocation](https://github.com/kubernetes/kubernetes/issues/18982), so this form of authentication should not be used where other options exist. Whilst it is likely not possible to completely eliminate client certificate authentication, it should be avoided for user authentication.

* If the cluster provides a client certificate user as part of initial setup, this user should *not* be used for general administration, instead it should be removed from the Kubernetes servers and stored in a secrets management system where it can be used in the event of a "break glass" situation.

* Access to the CertificateSigningRequest (CSR) API in Kubernetes should be restricted to only specific cases (e.g. Kubelet certificate rotation) to avoid users generating and approving new client certificates. Where access to this API is required, it should be audited and reviewed regularly.

* In unmanaged Kubernetes, access to the signing key should be very carefully controlled and audited (these files typically live with the Kubernetes configuration files).

For managed Kubernetes the picture of whether this feature is available is mixed. In Microsoft AKS it's possible to get a client certificate issued (indeed that's the default) and the CSR API is available. In Google GKE the first user doesn't use client certificates but the CSR API is available. In Amazon EKS, the first user is not a client certificate and the CSR API does not work for issuing new user accounts ([this may or may not be a bug](https://github.com/aws/containers-roadmap/issues/1604#issuecomment-1089918625), it's undocumented)

### Service Account Tokens

[Service accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) are used by Kubernetes workloads to authenticate to the Kubernetes API server where needed. Somewhat unusually these are provided to every workload by default, so operators need to actively disable them if not required. 

In older versions of Kubernetes (up to 1.24) by default the service account tokens were based on Kubernetes secrets. These tokens did not expire and cannot be revoked without deleting the service account they were associated with. In 1.24+ the tokens used by service accounts are based on Kubernetes TokenRequest API. These tokens have an expiry but still require the object they are associated with to be deleted for revocation.

So in terms of managing these tokens as close as possible to the PCI guidance there's a couple of recommendations

* Don't mount service account tokens into cluster workloads unless specifically required. 

* Where service account tokens are required, make use of the TokenRequest API and ensure that token lifespan is as short as practical.

Past these specific recommendations to do with Kubernetes defaults, recommendations will be specific to the Kubernetes distribution handles authentication.

## Section 1.4

**Threat** - Credentials used to access administrative accounts for either containers or container orchestration tools are stored insecurely, leading to unauthorized access to containers or  sensitive data.

**Best Practice** - Authentication mechanisms used by the orchestration system should store credentials in a properly secured datastore.

**Details** - There's a couple of places where Kubernetes credentials might be stored insecurely. The first relates to the [static token file](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#static-token-file) authentication option that Kubernetes provides. This isn't (in my experience) widely used, but it is an option. A cluster using this option stores tokens in clear text on the Control plane nodes of the cluster

* Static token authentication should not be used.

However there's another place where credentials can effectively be stored in clear on disk, and that's the more commonly used client certificate authentication option. Control plane nodes will have private keys for the API server and certificate authority held in unencrypted format, and node will have Kubelet private keys. This is pretty unavoidable so in general the goal here is to minimize access to them and audit any access that does occur.

* Access to Kubernetes X.509 key files should be restricted to authorised administrative users.

## Section 1.5

**Threat** - Availability of automatic credentials for any workloads running in the cluster. These credentials are susceptible to abuse, particularly if given excessive rights.

**Best Practices** - a. Credentials for the orchestration system should only be provided to services running in the cluster where explicitly required. b. Service accounts should be configured for least privilege. The level of rights they will have is dependent on how the cluster RBAC is configured.

**Details** - This recommendation strays a little into authorization but it's basically looking to address service account token security when applied to Kubernetes. As we mentioned earlier Kubernetes, by default, will give every workload in the cluster a service account token which can be used to access the Kubernetes API server. This can lead to security problems as they can end up with excessive access if there's a mistake made in RBAC configuration on the cluster. For example I've seen cases where installing a 3rd party product to a cluster adds `cluster-admin` rights to the `default` service account token in the `default` namespace. This meant that every other workload in that namespace could get `cluster-admin` rights! So there's a couple of Kubernetes recommendations for this section :-

* Ensure that `automountServiceAccountToken: false` is set on every service account and pod unless they are specifically required.
* Avoid using the `default` service account token, each workload that needs Kubernetes API server access should be provided a specific service account
* Ensure that Service account tokens are not granted excessive privileges. Review manifests that give them rights.

## Section 1.6

**Threat** - Static credentials i.e., passwords used by administrators or service accounts are susceptible to credential stuffing, phishing, keystroke logging, local discovery, extortion,  password spray, and brute force attacks.

**Best Practice** Interactive users accessing container orchestration APIs should use multi-factor authentication (MFA).

**Details** - From a Kubernetes perspective a recommendation to use MFA for administrative access essentially requires the use of external authentication for any production cluster. For managed Kubernetes clsuter this would generally lead to the use of the cloud IAM service provided by the CSP (e.g. AWS, GCP, Azure) and ensuring that MFA is setup there. For on-premises clusters something like OIDC authentication integrated with an enterprise IAM solution with MFA would be used.

## Conclusion

Authentication is one of the more challenging aspects of Kubernetes security as it's not something the open source project focuses on heavily, with the expectation that distribution/service providers can add suitable additional controls. There are some definite areas to be aware of though as in-built authentication methods are often still available even when a more secure alternative has been provided.

Next time, we'll move on to section 2 of the PCI guidance, on [authorization](https://raesene.github.io/blog/2022/10/08/PCI-Kubernetes-Section2-Authorization/).