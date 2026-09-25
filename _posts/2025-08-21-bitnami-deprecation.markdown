---
layout: post
title: "Bitnami Deprecation"
date: 2025-08-21 12:30:00 +0100
comments: false
categories: Containers
---

**Update** Looks like Bitnami decided to take some more time over this [details here](https://community.broadcom.com/tanzu/blogs/beltran-rueda-borrego/2025/08/18/how-to-prepare-for-the-bitnami-changes-coming-soon) and have some 1-day brown outs before removing the repos on Sept 29.

One constant of modern development environments is the ever increasing number of dependencies, and the problems that come when they get disrupted. Next week there could be a serious disruption in the container image ecosystem as a provider of popular images and helm charts changes their availability and tags.

## What's Happening?

[This Github issue](https://github.com/bitnami/charts/issues/35164) has most of the details, but it's a little hard to work out the exact impact from it. The TL;DR. is that Bitnami are moving from freely available images under the Docker Hub Username `bitnami` to a split of commercially maintained images under `bitnamisecure` and unmaintained legacy images under `bitnamilegacy`.

The exact timing is unclear as the issue mentions `gradually move existing ones` to the legacy repository, however the impact is going to start in a week's time starting August 28th 2025, so it's clear that organizations using these images will need to take action sooner rather than later.

## So what's the impact?

Well if you're either directly using images from `bitnami`, Helm charts that reference those images, or images that are built off those base images, you need to start using different images pretty quickly or you might find deploys or image builds failing.

## How big of a problem is this?

After reading this, I thought it could be worth looking at how many pulls these images are getting. Luckily Docker Hub provides pull statistics via their API, so by looking at changes over time we can get a reasonable idea of how many people are going to be affected.

Looking at pull statistics for popular bitnami images over the course of 6 days we can see that the most popular image `kubectl` got 1.86M pulls in that time period, and a large number of images have had over 100K pulls in that time, so it seems like these images are pretty heavily used.

![bitnami stats]({{site.url }}/assets/media/bitnami-stats.png)

## Conclusion

I've long said that, when using container images in production, it's vitally important that you build and maintain all of your own images, or if you want have some kind of commercial maintenance agreement for them. Relying on freely provided externally managed images is a recipe for problems down the line.

For now though, the critical point is that everyone using Bitnami images, needs to go and review all their usage and make a fairly rapid plan to address the risk of them breaking in the near future.