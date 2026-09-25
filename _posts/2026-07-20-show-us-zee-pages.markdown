---
layout: post
title: "Show us Zee Pages"
date: 2026-07-20 14:00:00 +0100
comments: false
categories: Kubernetes
---

Full disclosure, this is one of those posts I do where there's no dramatic pay off at the end, it's just documenting something I found that took me a while to figure out and that I thought was interesting :)

Understanding the running state of a Kubernetes cluster and its various components, has always been a bit of a tricky affair, with many configuration files and command line flags that need to be read and orders of precedence understood to get the final state. It's a topic that I've had a particular interest in, as I help maintain the [CIS benchmark for Kubernetes](https://www.cisecurity.org/benchmark/kubernetes), and part of that is explaining how to check cluster state in the interest of security hardening. Explaining the different options of where configuration can be loaded from is a bit tricky, so any option for deriving that information easily would be very welcome!

A relatively recent development in this space is the official addition of [z-pages](https://kubernetes.io/docs/reference/instrumentation/zpages/) to various Kubernetes components (although as we see there are some longer standing less official ones too). These are endpoints intended for cluster administrators to use for debugging and maintaining their cluster, which sounds ideal. However the reality is unfortunately a little more complex.

## The different types of z-pages

Looking at the [documentation on the Kubernetes website](https://kubernetes.io/docs/reference/instrumentation/zpages/), you'll see mention of two main z-pages, `statusz` and `flagz`

`statusz` is fairly straightforward, it returns the status of the component. As an example, you can see that endpoint on the kube-apiserver component with `kubectl get --raw /statusz`

```bash
apiserver statusz
Warning: This endpoint is not meant to be machine parseable, has no formatting compatibility guarantees and is for debugging purposes only.

Started:  Mon Jul 20 06:14:02 UTC 2026
Up:  3 hr 22 min 42 sec
Go version:  go1.26.2
Binary version:  1.36.1
Emulation version:  1.36
Paths:  /flagz /healthz /livez /metrics /readyz /statusz /version
```

One of the handy points there is it shows us which other resources available, with some newer ones like `flagz` and others which have been around a long time like `version`

Things get a little more complex if you look at `statusz` from another Kubernetes component like the kubelet. The first problem (depending on version and distribution) is you might not be able to get that endpoint via the Kubernetes API proxy, and the second is when you do get the information by directly querying the kubelet API, there's a new endpoint `configz`! 

```json
{
  "kind": "Statusz",
  "apiVersion": "config.k8s.io/v1beta1",
  "metadata": {
    "name": "kubelet"
  },
  "startTime": "2026-07-20T06:14:01Z",
  "uptimeSeconds": 16625,
  "goVersion": "go1.26.2",
  "binaryVersion": "1.36.1",
  "emulationVersion": "1.36",
  "paths": [
    "/configz",
    "/debug",
    "/flagz",
    "/healthz",
    "/metrics"
  ]
}
```

To get a full picture, after going round the core components, we can see that the set of available z-page endpoints varies depending on the service.

- **kube-apiserver** - /flagz, /healthz, /livez, /metrics, /readyz, /statusz, /version
- **kube-controller-manager** - /configz, /flagz, /healthz, /metrics
- **kube-scheduler** - /configz, /flagz, /healthz, /livez, /metrics, /readyz
- **kube-proxy** - /configz, /flagz, /healthz, /metrics, /proxyMode
- **kubelet** - /configz, /debug, /flagz, /healthz, /metrics

For the purposes of our original goal, the two important ones here are `flagz` and `configz` as they serve subtly different but similar purposes.

## Comparing flagz and configz

From this [blog article about z-pages](https://kubernetes.io/blog/2025/12/31/kubernetes-v1-35-structured-zpages/) `flagz` "Shows all command-line arguments and their values used to start the component (with confidential values redacted for security)". `configz` doesn't have an official definition AFAIK (it was not introduced in the general z-pages feature, it's been around since ~2016, as mentioned in [this PR](https://github.com/kubernetes/kubernetes/pull/20528/changes)) and its never been got to fully released as a feature, but in general it's designed to show the configuration of the component including parameters loaded from configuration files.

So... why does this difference matter? Well, if you're trying to debug or audit a cluster, and you're interested in the actual running configuration of a component, and getting that exact information can be a bit tricky.

*If* the component is configured purely with command line flags, then `flagz` will work just fine, however if there are any configuration files involved (and there very commonly are), then things get more interesting as we have multiple sources of configuration and also the option of default settings (if no explicit configuration is provided). At that point relying on `flagz` is a bad idea as it will return incomplete or downright inaccurate information.

Looking at a practical example of how configuration is derived, let's take the kubelet :-

From the [order of precedence for Kubelet flags](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/#kubelet-configuration-merging-order) an actively set command line flag will overwrite a value from the config file, so that's the highest priority.

Next if no command line parameter is provided and a config file is in use which specifies the parameter, that wins.

Then if a config file is provided but doesn't specify the explicit parameter, the default from the config file setup wins. 

Finally if there is no config file at all and no explicit CLI flag, the default from the CLI flags takes effect.

One thing that is important here is that the defaults specified in the configuration file specification are sometimes *different* from the default specified for command-line flags. For example, the kubelet's read only port defaults to enabled (10255/TCP) based on the CLI flags default and disabled (0) based on the config file default.

## Authentication and Authorization for configz

Given what we've just discussed, in most cases you'll want to rely on `configz` for information, as it's more complete than `flagz`. However that leads to our first problem which is kube-apiserver doesn't currently support `configz` :(

For the components it does support (kube-scheduler, kube-controller-manager, kube-proxy and the kubelet) we'll need to be able to authenticate to the service and have the appropriate authorization to get to the endpoint. There is an in-built clusterrole called `system:monitoring` that provides access to most of the z-pages buuuuut not `configz` :)

So for kube-scheduler and kube-controller-manager we need a clusterrole and clusterrolebinding that allow access to the `/configz` NonResourceURL, and we'll need network access to the localhost interface of the control plane node as those components only listen on that interface. Then for the kubelet, we'll need get rights to `node/configz` if fine grained Kubelet authorization is supported or, on older versions, get rights to `node/proxy` (N.B. that permission is [quite powerful](https://grahamhelton.com/blog/nodes-proxy-rce), so be careful).

Lastly there's the kube-proxy and interestingly this one doesn't need any credentials at all as the kube-proxy component doesn't support authentication. In *most* cases you'll need network access to the localhost interface of the worker node to get to that endpoint, but at least one distribution (AWS EKS) binds it to all interfaces so you can get to that from anywhere on the container network.

## Dealing with gaps in the information returned from configz

Once you've got your information, you might note that it doesn't cover every single supported parameter. This is because `configz` will not always report a value where the default is in use. So you can either make the assumption that if it's not returned then the **config file default** (as opposed to the CLI default) applies, or you can add some logic that fills in the gaps.

## zeedumper

To save some time for people doing this manually I've got a little utility called [zeedumper](https://github.com/raesene/zeedumper/) that I've been workin on, which can drop out the information from `flagz`, `statusz`, and `configz` from each component. N.B. This isn't anywhere near a v1 and it does create a pod and some rights in the cluster, so don't run it in production without testing first :) 

zeedumper will try and fill in the missing default information for `configz` to give a complete picture for the kubelet and kube-proxy components (kube-scheduler and kube-controller-manager still to come)

## Conclusion

While z-pages are quite a useful idea for cluster troubleshooting and configuration review, there's unfortunately quite a lot of details to keep in mind and pitfalls to be aware of. I hope that as time goes on the picture will get simpler and we'll get a unified cluster configuration endpoint as it'll make a variety of cluster administration tasks quite a lot easier.

