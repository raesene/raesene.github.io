---
layout: post
title: "A Census of Kubernetes Clusters" 
date: 2021-06-05 15:00:39 +0100
comments: false
categories: Kubernetes
---

I've always been a fan of spelunking around in Internet data sets to see what interesting things I can find, so when I saw that [Censys](https://censys.io/) had a new version of their search site, and that it had classified data about Kubernetes clusters, I had to have a look around :) Before I talk about the results, I will say I was very impressed by their new offering, the search speed is amazing for the amount of data and there's some really useful search filters, some of which I'm using here.

So with access to the data my goal was to see what information I can find out about Internet facing Kubernetes clusters. There are quite a lot of these around mainly because a lot of managed Kubernetes vendors have defaulted to making the API server Internet accessible by default (top security tip is, turn that off if you don't need it!)

## Finding Kubernetes Servers on the Internet

So our first question is, "how do we know we've found a Kubernetes server?". The first option takes advantage of a peculiarity of how Kubernetes does TLS certificates. In a standard cluster, the API server will have a number of Subject Alternative Name fields which are predictable and consistent. For example, if I dump the API server TLS cert from a kind cluster, I get the following in the SAN field

```
DNS:kind-control-plane, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, DNS:localhost, IP Address:10.96.0.1, IP Address:172.18.0.2, IP Address:127.0.0.1
```

Names like `kubernetes.default.svc.cluster.local` have been present in pretty much every cluster I've looked at, so are probably a reasonable place to start. Censys allows you to search for names in a cert, so with a search like [this](https://search.censys.io/search?resource=hosts&q=services.tls.certificates.leaf_data.names%3D%22kubernetes.default.svc.cluster.local%22) we can get some results.

At the time of writing that's returning 776795 results, a fair few clusters! This leads on to another question though which is, What versions of Kubernetes are running here?

## Using Kubernetes Unauthenticated Version Endpoint

An interesting point about the Kubernetes API server is that, by default, many clusters will make the `/version` endpoint available on the API server without authentication. This endpoint returns specific information like the specific version of Kubernetes in use and the version of golang used when compiling it. Another top security tip is, disable this if you don't need it!

However fortunately for our census, many Kubernetes distributions don't disable this option. In the past I've considered scanning the Internet for this info., so I was very happy to find out that Censys had done the hard work for me, and it's available as a query in their new search API.

[This Search](https://search.censys.io/search?resource=hosts&q=services.kubernetes.version_info.git_version%3D%22%2A%22) will give us a list of all the servers which return that information. At just over 90,000 results, that's not a bad place to start.

We can then use the very handy "report" feature to automatically extract all the versions and a count for each one. The report can be seen [here](https://search.censys.io/search/report?resource=hosts&q=services.kubernetes.version_info.git_version%3D%22*%22&field=services.kubernetes.version_info.git_version&num_buckets=1000). From that we can see a couple of interesting points immediately. The first one is that many of the clusters have customized version strings, due the distribution vendor modifying the API server binary. Fortunately most of the vendors put a noticable pattern (e.g. 'gke' for GKE) so we can form some conclusions about which distributions we're seeing.

## Splitting down the Distributions

Looking through the identified distributions, the two largest are GKE and EKS, somewhat unsurprisingly. Based on a lack of any AKS distributions, it looks like Microsoft are blocking access to the version endpoint by default.

![Distributions]({{site.url }}/assets/media/kubernetes-distros-censys.png)

In addition to the big 3 managed distributions, I noticed quite a lot for IKS (IBM's managed Kubernetes) and OpenShift. Worth noting if you're looking at the data, OpenShift doesn't include their name in the version string but have a tell-tale pattern with a hex string at the end (e.g. `v1.9.1+a0ce1bc657`)

## What Versions are In Use?

The next interesting question is, what Kubernetes versions are in use? A bit of data manipulation is needed to get the major version from each running cluster, but at the end of it we end up with a chart like this

![Distributions]({{site.url }}/assets/media/Kubernetes_versions-censys.png)

First up, there's *a lot* of unsupported Kubernetes out there. Looking at the major distributions out there, most have dropped support for anything older than 1.17. Based on this, there's at least 26% of clusters running unsupported versions.

In particularly notable results, there's two clusters running Kubernetes 1.2! Both of them appear to be OpenShift clusters.

## What else is there to find?

I'm just getting started on the search for interesting things, but one thing I noticed by looking through the clusters, you can also find other interesting things. One point I noted was that there are currently over 600 systems running the famous Weave [Sock Shop](https://microservices-demo.github.io/) demonstration App. (Censys Search [here](https://search.censys.io/search?resource=hosts&q=%22WeaveSocks+Demo+App%22))


## Conclusion

Amusing results aside, there's a couple of important points which come out of this data. The first one is that it's quite easy to fingerprint Kubernetes clusters on the Internet, either via distinctive TLS certificates, or exposed API endpoints.

The second point is the number of companies running unsupported Kubernetes versions exposed to the Internet. These clusters are unlikely to receive security patches, so they could be a single CVE away from compromise.

In general, from a security standpoint, I'd very much recommend not exposing your API server directly to the Internet unless you really need to. And if you do, make sure you keep it running a supported version of the software!
