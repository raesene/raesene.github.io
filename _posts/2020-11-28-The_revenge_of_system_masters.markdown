---
layout: post
title: "The revenge of system:masters, return of the AKS" 
date: 2020-11-29 12:00:39 +0100
comments: false
categories: Kubernetes
---

When looking at an AKS cluster recently, I came across some unusual default behaviour, which I thought deserved some more investigation over the weekend. Seems like AKS is making some ... interesting... choices with regards to user authentication, in some setups.

Usually when you authenticate to an Azure AD (AAD) enabled AKS cluster it uses OAuth with a short renewal window for credentials, which is great as it leverages the Azure AD authentication process and has a nice short expiry to reduce the risk of a misplaced kubeconfig file. 

However there's an option on the `az aks get-credentials` command which acts a bit differently. If you pass `--admin` to that command, instead of using OAuth, it adds a client certificate file to your `kubeconfig`. 

Client certificate authentication in Kubernetes is a bit problematic for a number of reasons, the main one being that there is **no way** to revoke a credential once it's issued (short of rolling the keys for the whole certificate authority), that means it's generally unsuitable for any production cluster and definitely production clusters used by large organizations.

However digging into the client cert issued by AKS, things get worse. If you decode it (the one below is from a sample AKS cluster I set-up using the standard [AKS QuickStart](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)), you'll see something like this

```bash
 Validity
   Not Before: Nov 28 20:11:59 2020 GMT
   Not After : Nov 28 20:21:59 2022 GMT
   Subject: O = system:masters, CN = masterclient
```

The way that Kubernetes client certificate authentication works, the `CN` field is considered the username and the `O` field is the group the user belongs to, so this is logging in with a username of `masterclient` and a group of `system:masters`.

First problem is the username. That's not the name of the user I used to login, this is a generic username. That means that any actions taken by this credential will show up with a generic username not matter who uses it, bit of a problem for auditing.

The second problem is the group name. `system:masters` is a special group in Kubernetes, its access as a cluster-admin is hardcoded into the Kubernetes source code and there's no way to remove those rights, so this credential will always have cluster-admin rights.

The third problem is the expiry, which is 2 years from cluster setup. So anyone with this credential has a generic cluster-admin level login which can't be revoked (short of rolling the entire certifiicate authority used for the cluster) and can't be audited, which lasts for up to two years.

Another surprising thing I noticed was that, in non-AAD enabled clusters, users with the `Azure Kubernetes Service Cluster User Role` get this same certificate given to them when they use `az aks get-credentials` even without the `--admin` switch! This is a [known issue](https://github.com/Azure/AKS/issues/1343) with AKS...

## Conclusion

A couple of takeaways if your organization is planning to use Azure AKS. First up if your clusters are AAD enabled, be **very** careful with who has rights to used the `--admin` switch on the `get-credentials` command. Secondly, I'd recommend not exposing your clusters directly on the Internet to reduce the likely impact of this.

If you're using non-AAD enabled clusters, it's worth noting that all your users may be using generic cluster-admin credentials, so you should check the generated kubeconfig files and plan for that.