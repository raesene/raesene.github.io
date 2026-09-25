---
layout: post
title: "A final Kubernetes census" 
date: 2024-02-17 15:27:00 +0100
comments: false
categories: Kubernetes
---

Well, all good things must come to an end. Over the last couple of years I've been using the [Censys](https://censys.io) API to track the number of Kubernetes clusters exposed to the internet which disclose their version number, and I've written about it a couple of times [here](https://raesene.github.io/blog/2021/06/05/A-Census-of-Kubernetes-Clusters/) and [here](https://raesene.github.io/blog/2022/07/03/lets-talk-about-kubernetes-on-the-internet/)

After a couple of failures of the daily script to run, I logged into my Censys account to see a banner saying that free access to their API had been removed.

![No free Censys]({{site.url }}/assets/media/censys-no-free-api.png)

Looking at the plans available they were a bit pricey for this hobby project, so I've decided to stop the daily script and this will be the last post on the topic.

## Kubernetes numbers

So what's the final outcome? Well the last scan shows 1,626,249 cluster hosts with visible version numbers on the Internet (and it's worth noting the final number will be higher as some distributions like AKS don't expose version number without authentication). Compared to 842,350 hosts in August 2022 when this dataset started, that's a pretty significant increase (I've got data from earlier than that in the posts above but Censys changed their scanning methodology in August 2022, so it's not directly comparable).

In terms of visible versions, the most common major version is v1.26, which is reasonably up to date, but still quite a way back from the latest released version (v1.29). There is a "long tail" quite visibly present, so it's obvious that some cluster operators are finding the update cycyle challenging.

Looking at a graph of all the versions we can see the different versions and how they've changed over time.

![K8s versions]({{site.url }}/assets/media/kubernetes-versions-2024.png)

## Conclusion

This has been a pretty interesting project for providing some insights into how Kubernetes adoption runs over time and what versions of Kubernetes are actually in use. It's amusing that it was enabled by a quirk of Kubernetes default configuration (exposing `/version` without authentication) and defaults from the major managed Kubernetes distributions (which put the API server on the Internet by default).

The data is available at [this repo](https://github.com/raesene/public-k8s-censys) along with some details of how it was analysed, so that might well be useful for someone else :)