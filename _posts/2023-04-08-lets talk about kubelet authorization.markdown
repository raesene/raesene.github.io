---
layout: post
title: "Let's talk about Kubelet authorization" 
date: 2023-04-08 15:27:00 +0100
comments: false
categories: Kubernetes
---

I've been meaning to write a post about Kubelet authorization for a while now, and as there have been some posts this week where it got a mention, now seems like a good time!

The Kubelet is the Kubernetes component which runs on each worker (and possible control plane) node and is responsible for managing the container runtime on the host. It communicates with the Kubernetes API server to get information about workloads that should be running on the node and then instantiates them using a container runtime like Containerd. To do this, it obviously needs credentials to access the API server, and needs rights to things like pods and also associated objects like secrets.

From a security perspective these Kubelet credentials are important as, if an attacker breaks out from a container to the underlying node, there will generally be a set of kubelet credentials available to them, so they could be used to escalate rights to the cluster, as a result it's been necessary for the Kubernetes project to take steps to restrict what the Kubelet can do, to reduce the risk of privilege escalation.

## A Brief aside - Kubernetes authorization modes

An important aspect of Kubernetes to discuss before talking about exactly how Kubelet authorization works, is how Kubernetes generally handles authorization. Whilst most clusters will use RBAC, it's possible to have multiple authorization modes in any cluster. Rights provided by each authorization mode are cumulative, so it's important to be careful about inadvertently granting rights to users. Also, if you're using Kubernetes in-built tooling for listing all of a users permissions (via the `kubectl auth can-i --list` command showne below), it's worth noting that this *only* works with RBAC, rights granted via other authorization modes will not be analyzed, although you can check for individual rights granted via any authorization mode using the `kubectl auth can-i` command.

## Kubelet authorization

Having talked about the fundamentals, let's look at how Kubelet authorization works. We'll start with a [KinD](https://kind.sigs.k8s.io/) cluster and look at what's visible there.

For this we'll use a cluster with two worker nodes, using a simple kind config

```bash
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

Then we can start the cluster up with `kind create cluster --config kind-config.yaml --name kubeletauthz`. Once the cluster is up and running we can shell into one of the worker nodes to look at the Kubelet credentials. You can find out what Kubeconfig file the Kubelet is using by looking at the parameters passed to the Kubelet on the command line. In the case of Kubeadm the default will be `--kubeconfig=/etc/kubernetes/kubelet.conf`. Once we know the location we can use that with Kubectl, for example to see a listing of pods in the cluster

```bash
kubectl --kubeconfig=/etc/kubernetes/kubelet.conf get po -A
NAMESPACE            NAME                                                 READY   STATUS    RESTARTS   AGE
kube-system          coredns-787d4945fb-4rspx                             1/1     Running   0          4m43s
kube-system          coredns-787d4945fb-p5g6f                             1/1     Running   0          4m43s
kube-system          etcd-kubeletauthz-control-plane                      1/1     Running   0          4m56s
kube-system          kindnet-clmjl                                        1/1     Running   0          4m43s
kube-system          kindnet-jhngj                                        1/1     Running   0          4m26s
kube-system          kindnet-kjsvt                                        1/1     Running   0          4m27s
kube-system          kube-apiserver-kubeletauthz-control-plane            1/1     Running   0          4m55s
kube-system          kube-controller-manager-kubeletauthz-control-plane   1/1     Running   0          4m55s
kube-system          kube-proxy-9v62x                                     1/1     Running   0          4m43s
kube-system          kube-proxy-q8t5c                                     1/1     Running   0          4m26s
kube-system          kube-proxy-vnrt6                                     1/1     Running   0          4m27s
kube-system          kube-scheduler-kubeletauthz-control-plane            1/1     Running   0          4m56s
local-path-storage   local-path-provisioner-75f5b54ffd-52xm9              1/1     Running   0          4m43s
```

Usually to check the rights of a principal in Kubernetes we'd use the command `kubectl auth can-i --list` and if we try that with the Kubelet credentials we get back something like this.


```bash
kubectl --kubeconfig=/etc/kubernetes/kubelet.conf auth can-i --list
Warning: the list may be incomplete: node authorizer does not support user rule resolution
Resources                                                       Non-Resource URLs   Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io                   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io                    []                  []               [create]
certificatesigningrequests.certificates.k8s.io/selfnodeclient   []                  []               [create]
                                                                [/api/*]            []               [get]
                                                                [/api]              []               [get]
                                                                [/apis/*]           []               [get]
                                                                [/apis]             []               [get]
                                                                [/healthz]          []               [get]
                                                                [/healthz]          []               [get]
                                                                [/livez]            []               [get]
                                                                [/livez]            []               [get]
                                                                [/openapi/*]        []               [get]
                                                                [/openapi]          []               [get]
                                                                [/readyz]           []               [get]
                                                                [/readyz]           []               [get]
                                                                [/version/]         []               [get]
                                                                [/version/]         []               [get]
                                                                [/version]          []               [get]
                                                                [/version]          []               [get]
```

Notably there are no rights to the `pod` objects that we just looked at! The clue to what's going on here is the `Warning` line at the top which notes that the `node authorizor` doesn't support user rule resolution.

One quick aside is that you may be confused by the Kubelet not using RBAC as there is a `clusterrole` called `system:node` which looks like it would provide rights to nodes, however the corresponding `clusterrolebinding` doesn't actually have any subjects (which is weird), so it has no effect! 

## Node authorizer

The key to what's going on here can be see in the configuration of the Kubernetes API server. If you look at the parameters passed to the `kube-apiserver` component you'll see this stanza `--authorization-mode=Node,RBAC` indicating that there are two modes of authorization configured and, as we said earlier, the rights from these are cumulative.

The [Node authorization mode](https://kubernetes.io/docs/reference/access-authn-authz/node/) is an authorization mode with one purpose which is to provide rights to Kubelets. From the documentation page we can see that this authorization mode allows access to the kind of resources that the Kubelet needs to use like `pods`, `nodes`, `configmaps` and `secrets`. Within that group it also needs to restrict *which* secrets etc are actually accessible as you don't want a Kubelet on one node to be able to access secrets intended for pods running on another node.

The exact logic of what is allowed can be seen in [the code](https://github.com/kubernetes/kubernetes/blob/bd190762fb77b78788e1a3ba72d70a8a08e184b8/plugin/pkg/auth/authorizer/node/node_authorizer.go#L41)

```bash
// NodeAuthorizer authorizes requests from kubelets, with the following logic:
// 1. If a request is not from a node (NodeIdentity() returns isNode=false), reject
// 2. If a specific node cannot be identified (NodeIdentity() returns nodeName=""), reject
// 3. If a request is for a secret, configmap, persistent volume or persistent volume claim, reject unless the verb is get, and the requested object is related to the requesting node:
//    node <- configmap
//    node <- pod
//    node <- pod <- secret
//    node <- pod <- configmap
//    node <- pod <- pvc
//    node <- pod <- pvc <- pv
//    node <- pod <- pvc <- pv <- secret
// 4. For other resources, authorize all nodes uniformly using statically defined rules
```

You can see this in effect if you try to get `secrets` from a cluster with Kubelet credentials

```bash
kubectl --kubeconfig=/etc/kubernetes/kubelet.conf get secrets -A
Error from server (Forbidden): secrets is forbidden: User "system:node:kubeletauthz-worker" cannot list resource "secrets" in API group "" at the cluster scope: can only read namespaced object of this type
```

One important point to note on this is where the logic is "reject", this just passes the request to other configured authorization modes, so if cluster RBAC has been modified to allow the `system:nodes` group to do something in excess of what the Node authorization mode allows then that will still be allowed.

You can see this by, for example, editing the `system:node` `clusterrolebinding` to add the `system:nodes` group as a subject, by adding these lines to it.

```bash
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
```

Once you've done that, if you use Kubelet credentials to try and get secrets at a cluster level, it works fine :)

```bash
kubectl --kubeconfig=/etc/kubernetes/kubelet.conf get secrets -A
NAMESPACE     NAME                     TYPE                            DATA   AGE
kube-system   bootstrap-token-abcdef   bootstrap.kubernetes.io/token   6      34m
```

There are a couple of places where this mode can't effectively restrict permissions, which are in node and pod properties, for that we need another component.

## NodeRestriction Admission Controller

This is where a specialized admission controller comes in. The [NodeRestriction admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) looks at requests from Kubelets and, where they relate to `pods` and `nodes` it limits the rights to only those that are appropriate for the Kubelet. For example it restricts what properties of `node` objects can be modified, to stop it changing it's own security classification, for example.

## Variations in Kubernetes distributions

It's important to note that, as with most Kubernetes configuration topics, what we've discussed here relates to vanilla Kubernetes, in this case Kubeadm. Distribution providers are free to change this configuration and indeed some do so. For example Azure AKS currently defaults to allowing Kubelets `GET` access to `secrets` at the cluster level!

## Conclusion

When looking at Kubernetes authorization it can be tempting to focus purely on RBAC as it's the most common option deployed in Kubernetes clusters today. However as we've seen there are times when RBAC alone won't provide an adequate level of security and supplemental authorization modes and admission controllers are required. In this post we've looked at the Node authorization mode and NodeRestriction admission controller which are used to provide rights to Kubelets to access the resources they need to function.


