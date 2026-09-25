---
layout: post
title: "Let's talk about Kubernetes on the Internet" 
date: 2022-07-03 09:10:39 +0100
comments: false
categories: Kubernetes
---

There's been a couple of studies recently released by security research companies about exposed Kubernetes clusters on the Internet, and whilst it's nice to see the security industry focusing a bit more on Kubernetes, some of the analysis misses some of the details of why Kubernetes clusters are exposed to the Internet and what some of the results mean, so I thought it would be a good opportunity to [revist](https://raesene.github.io/blog/2021/06/05/A-Census-of-Kubernetes-Clusters/) this topic, also as there have been some developments in what information can be found via Internet search engines.

## Background - Kubernetes Network footprint

Kubernetes is made up of a number of REST APIs which are commonly exposed on network ports. We've got

- The [Kubernetes API server](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/). commonly exposed on 443/TCP, 6443/TCP or 8443/TCP.
- The [Kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/). Commonly exposed on 10250/TCP and (in older clusters) 10255/TCP
- [Etcd](https://etcd.io/) - Commonly exposed on 2379/TCP and 2380/TCP

There's also an additional set of listening ports but in *most* clusters these will be bound to localhost, so won't show up in our Internet scanning

- [Kubernetes Controller Manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/). Defaults to 10257/TCP in recent Kubernetes version.
- [Kubernetes Scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/). Defaults to 10259/TCP in recent Kubernetes versions.
- [Kube-Proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/). Defaults to 10256/TCP.

So from an Internet scanning perspective we've got a range of ports to look for. The API Server, Kubelet and etcd are also likely to be the most significant from a security standpoint, if they're mis-configured, so it makes sense to focus there.

## Finding Kubernetes Clusters

So now we know the network ports and services we're looking for, how can we reliably identify that what we're talking to is a Kubernetes service?

### Kubernetes API Server 

#### TLS Certificate Information

Kubernetes API servers are secured by TLS and as they are contacted by clients both inside and outside the cluster, they need to make sure they have name fields which match the various ways they can be contacted. This is handled by putting information into the Subject Alternative Name field in their certificates.

If we look at a standard [Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) cluster we can see some of that information.

```
 ssl-cert: Subject: commonName=kube-apiserver
| Subject Alternative Name: DNS:kubeadm2nodemaster, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:10.96.0.1, IP Address:192.168.41.77
| Issuer: commonName=kubernetes
```

From this there's a couple of points that are worth noting. Firstly internal cluster IP addresses are leaked (useful for Internet based attackers doing reconnaissance), secondly there are names that will typically appear on *every* Kubernetes cluster, making identification easy. `kubernetes.default.svc.cluster.local` is the DNS name used for clients inside the cluster network to connect to the API server so will be present in pretty much every cluster.

#### Response codes

In addition to the TLS certificate information we can also tell some things about an API server based on the response codes we get. For most clusters, if you try to curl the root path of the API server you'll get a response something like this

```
curl -k https://192.168.41.77:6443
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```

What this indicates is that we are being authenticated to the API server (so anonymous authentication is enabled, which is the default) but that we're not authorized to get that URL. You can see from the response that we've been assigned the username `system:anonymous` which is given to any requests made without other credentials.

Whilst 403 responses are the most common from the API server, a Kubernetes API server will respond with a 401 (unauthorized) instead. The most obvious one would be if anonymous authentication is disabled on the API server. Also if HTTP basic authentication (only available till 1.19) or token authentication are enabled and an incorrect set of credentials provided a 401 will also be returned.

Obviously there's another option for response codes which is 200. This would occur for a path which is permitted for the `system:anonymous` user or `system:unauthenticated` group. In default Kubernetes there are still a couple of paths that this'll work for. Most notably for the purposes of fingerprinting Kubernetes clusters `/version` will generally be visible. Requests to that path will return something like this, which provides quite a bit of information about the running software.

```json
{
  "major": "1",
  "minor": "21",
  "gitVersion": "v1.21.2",
  "gitCommit": "092fbfbf53427de67cac1e9fa54aaa09a28371d7",
  "gitTreeState": "clean",
  "buildDate": "2021-06-16T12:53:14Z",
  "goVersion": "go1.16.5",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

### Kubelet

Typically identifying the Kubelet is relatively straightforward as there aren't a large number of common services which use 10250/TCP and/or 10255/TCP.

#### Kubelet Response Codes

Usually the Kubelet will return a 404 response when the root path it queried. This indicates that anonymous authentication is enabled (which is the default) but that it doesn't have anything present at that URL. Querying a valid path for the API (e.g. `/pods`) will return a 401 unauthorized message, unless the `alwaysAllow` authorization mechanism is set-up, when that path would return a list of the pods on the node.

In the event that 10255/TCP is visible on a cluster (only older versions) this is the unauthenticated Kubelet "read-only" port which will return a detailed list of pods running on the node.

### etcd

Like the Kubelet etcd runs on a reasonably unusual set of ports (2379/TCP and 2380/TCP). A running etcd instance may not be related to a Kubernetes cluster as it can be used independently, but when seen along-side ports like 10250/TCP it's a fair bet it's support a Kubernetes installation.

When supporting Kubernetes, etcd will almost always be set-up with client certificate authentication only, so requests will just be rejected with a bad certificate error like this

```
curl: (35) error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
```

## Finding Kubernetes Clusters on the Internet

So now we know what we're looking for, how do we find it? Luckily, there are multiple Internet search engines which provide filters to make finding Kubernetes ports easy. [Shodan](https://shodan.io), [Censys](https://censys.io/) and [Binary Edge](https://www.binaryedge.io/) are all options. In addition to pre-packed filters we can also use certain features of how Kubernetes to identify services listening on the Internet (or any other network). 

Of the three Shodan currently finds the most servers, to let's look at some of the options to find things there. Some of the query results below will require a shodan account to look at so I've put screenshots for some of the more interesting information.

### Basic Information for exposed clusters

The most basic search available is one for [product:"Kubernetes"](https://www.shodan.io/search?query=product%3A%22Kubernetes%22). This currently returns 1.3M results. Of these ~240k are Kubelets, so that leaves us with a bit over a million likely Kubernetes API servers.

![shodan basic results]({{site.url }}/assets/media/shodan-product-kubernetes.png)

So the first question that likely occurs is "why are there so many Kubernetes services on the Internet?". Whilst there's a number of reasons, the major one comes down to the defaults used by the 3 major managed Kubernetes services, EKS, AKS and GKE. All of these services default to putting the API server directly on the Internet, and we can see evidence of this from Shodan's reporting of the netblock owners for the various exposed services.

Looking at the [organizations](https://www.shodan.io/search/facet?query=product%3A%22Kubernetes%22&facet=org) for this we can see the largest are Google, Amazon and Microsoft respectively and that they account for well over 600k of the exposed services.

![shodan org results]({{site.url }}/assets/media/shodan-kubernetes-orgs.png)

The next likely interesting point is around the versions of Kubernetes running on the exposed services. Shodan (like the other Internet search engines) pull that information out by querying the `/version` endpoint where it's exposed.

Looking at the [version information](https://www.shodan.io/search/facet?query=product%3A%22Kubernetes%22&facet=version) we can see a couple of interesting things. Firstly we see EKS and GKE versions there but not AKS. This is because, by default, both Amazon and Google make that endpoint available without authentication and Microsoft does not.

Looking at the top versions available the other interesting point is that, while most are recent'ish, they're still falling behind the latest available (1.24) and quite a large number of clusters are running unsupported versions.

![shodan version results]({{site.url }}/assets/media/shodan-exposed-versions.png)

### Response code variations

One of the things that was reported on by other research was around the response codes for Kubernetes API servers. Looking at the information on Shodan we can see something quite interesting about the split of responses. The majority of API servers respond with the 403 forbidden code as shown [here](https://www.shodan.io/search?query=product%3A%22Kubernetes%22+403) and looking at the responses for 401 unauthorized [here](https://www.shodan.io/search?query=product%3A%22Kubernetes%22+401) what we can see is that the vast majority of them are in Microsoft's netblock space. The likely explanation here is that Microsoft's AKS product is that they're disabling anonymous authentication to the API server or they're using some kind of additional load balancer or proxy in front of the API servers which is returning 401 to requests to the root path.

![shodan 401 response code results]({{site.url }}/assets/media/shodan-kubernetes-401.png)

### Kubelets

Searching for what we know about Kubelets, we can see [results for a response code of 404 on port 10250/TCP](https://www.shodan.io/search?query=product%3A%22Kubernetes%22+http.status%3A%22404%22+port%3A%2210250%22). It's kind of interesting that there are as many results as this as, while there's some reasons why you might want the API server directly connected to the Internet, there's not many reasons to directly expose the Kubelet.

![shodan kubelet results]({{site.url }}/assets/media/shodan-kublets-exposed.png)

### Etcd

Tracking down etcd hosts is a little more difficult. As, by default, they won't actually form a connection without a valid client certificate. We can look at Shodan's data for [port 2379](https://www.shodan.io/search?query=port%3A%222379%22). In there there's a subset of results for a product of etcd and looking at [the results](https://www.shodan.io/search?query=port%3A%222379%22+product%3A%22etcd%22) what's interesting is that these are essentially unauthenticated etcd services, which may or may not be related to Kubernetes clusters.

![shodan unauth etcd results]({{site.url }}/assets/media/shodan-unauth-etcd.png)


### Conclusion

The goal here was just to record a bit of information about Kubernetes network attack surface, some of the tricks of identifying Kubernetes clusters based on their responses to basic requests and look at what information is visible on the Internet relating to exposed Kubernetes services.

Possibly the most important point here is that if you're using one of the major managed Kubernetes distributions, you're possibly exposing more information than you want to via the exposed API server port, and if you can, it's a good idea to remove that exposure by restricting access to your cluster.