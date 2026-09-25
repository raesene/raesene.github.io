---
layout: post
title: "Let's talk about anonymous access to Kubernetes" 
date: 2023-03-18 15:27:00 +0100
comments: false
categories: Kubernetes
---

This week there were some articles about the [Dero Cryptojacking operation](https://www.bleepingcomputer.com/news/security/first-known-dero-cryptojacking-operation-seen-targeting-kubernetes/) and one of the details about what the attackers did caught my eye. It was mentioned that they were attacking clusters that allowed anonymous access to the Kubernetes API. Exactly how and why anonymous access is possible to Kubernetes is kind of an interesting topic that touches on a few different areas, so I thought I'd write a bit about it.

## How does anonymous access work?

Whether anonymous access works on a cluster is controlled by a flag on the [kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) component. The flag is `--anonymous-auth` and it defaults to `true`, so if you don't see it in the list of parameters passed to the server anonymous access will be enabled.

However on it's own that won't actually give attackers a lot of access to the cluster, as it only covers one of the three gates a request passes through before it's processed. As shown in the Kubernetes docs [controlling access](https://kubernetes.io/docs/concepts/security/controlling-access/) section after authentication, the request then has to pass authorization and admission control.

## Authorization and anonymous access

So the next step is that a request will need to match an authorization policy (typically RBAC, but possibly others as well). Of course in order to do that the request has to be assigned to an identity, and that's where the `system:anonymous` user and `system:unauthenticated` group come in. These identities are assigned to any request that doesn't have a valid authentication token, and are used to match against authorization policies.

You can see this in action by looking at the `system:public-info-viewer` `clusterrolebinding` on a Kubeadm cluster.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:public-info-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:public-info-viewer
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:unauthenticated
```

## Anonymous access in the wild

Now we know how anonymous access works, the question is going to be "how common is this?". The answer is that most major distributions will enable anonymous access by default and will generally provide access to some limited number of endpoints via the `system:public-info-viewer` `clusterrole` which provides access to the `/version` endpoint along with a couple of others.

To get a sense for how many clusters this applies to we can just use [censys](https://search.censys.io/) or [shodan](https://www.shodan.io/) to look for clusters that return version information. This [censys query](https://search.censys.io/search?resource=hosts&q=services.kubernetes.version_info.git_version%3D%22*%22) for example shows just over one million hosts that return version information, so we can say that it's a fairly common configuration.

A more serious question, which corresponds more to the points raised in the dero artice is, how many of these clusters would allow for an attacker to create workloads in them. Whilst you can't get that exact information from Censys, it does have a query showing clusters that allow for anonymous users to enumerate pods in the cluster which shows [302 cluster nodes](https://search.censys.io/search?resource=hosts&q=services.kubernetes.pod_names%3D%22*%22) at time of writing. I'd guess some/most of those are honeypots, but also probably a couple of vulnerable clusters in there.

## Disabling anonymous access

On an unmanaged cluster (e.g. Rancher, Kubespray, Kubeadm) you can disable anonymous access by passing the `--anonymous-auth=false` flag to the `kube-apiserver` component. On managed clusters (e.g. EKS, GKE, AKS) you can't do that, however what you can do is remove any RBAC rules which allow anonymous users to perform actions. For example, on a Kubeadm cluster you can remove the `system:public-info-viewer` `clusterrolebinding` and the `system:public-info-viewer` `clusterrole` and that effectively stop anonymous users getting information from the cluster.

Of course, if you have any applications that rely on those endpoints (for example for health checks) they would break, so it's important to test any changes you make to the cluster. One option here would be to review your audit logs and see if there are any anonymous requests made to the API server.

## Conclusion

Allowing some level of anonymous access is a common default in Kubernetes. In and of itself it's not a great security concern, however it does mean that in many configurations the only thing stopping attackers compromising your cluster is RBAC rules, so a single mistake could cause significant issues, especially if your cluster is exposed to the internet.