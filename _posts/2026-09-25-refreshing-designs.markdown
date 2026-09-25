---
layout: post
title: "Refreshing Designs"
date: 2026-09-25 17:00:00 +0100
comments: false
categories: LLMs
---

Like many techies, I've got a collection of websites for various projects I've worked on, or information I want to share and track. Unfortunately my abilities to make nice looking designs for those websites are .... limited. I've never had the greatest aesthetic sense and my front-end developments skills are not the best.

Now a lot of people have addressed this kind of gap with LLMs and tools like Claude design. It's fair to say they can create decent looking sites and things like presentations, but they do all tend to look a lot alike, which gets a bit boring.

So this morning when I came across a new website and approach to design I thought I'd try it out!

## Katagami, Opus 5.5, and images

The site I saw today is called [katagami](https://katagami.ai/) and it's got a wide range of designs available for websites. Interestingly it also has an MCP server that you can combine with coding agents, letting the agent send information about the site content to a service and receive back recommendations on which designs would work well.

So I decided to try out the recently released Opus 5.5 which thankfully seems **much** improved from Opus 5. Hooking it up to the Katagami MCP was simply a matter of providing the URL to the agent and it added the integration.

The next slight hurdle I came across was Opus' lack of image generation capabilities. While it can create things with SVG, it can't do standard image generation. Here My approach was to combine it with an [Openrouter](https://openrouter.ai/) subscription, so it could call any of the image generation models available via that gateway, and use that to create the images called for by the designs.

With all of that hooked up, I simple opened each repository and asked for a new design, reviewed candidates and chose my favourite and then let it go ahead and create the design. As all the sites I have are github pages sites built with Jekyll adding new designs is a relatively straightforward process.

One place where this exceeded my expectations was that it didn't just to a straight change of CSS/JS but actually re-organized the sites as well, generally for the better.

## Results

Of course the proof of the pudding is in the eating, so what did it come up with

### [www.mccune.org.uk](https://www.mccune.org.uk) - Personal portfolio site

![www.mccune.org.uk homepage]({{site.url }}/assets/media/redesign-mccune.jpg)

### [raesene's Ramblings](https://raesene.github.io) - Blog

![raesene's Ramblings homepage]({{site.url }}/assets/media/redesign-ramblings.jpg)

### [Cloud Native Security Talks](https://talks.container-security.site) - Archive of security talks from Kubecons

![Cloud Native Security Talks homepage]({{site.url }}/assets/media/redesign-talks.jpg)

### [Container Security Site](https://container-security.site) - Information about container security

![Container Security Site homepage]({{site.url }}/assets/media/redesign-container-security.jpg)

### [Dearbhadh](https://raesene.github.io/dearbhadh) - Results of a Kubernetes Security benchmark for LLMs

![Dearbhadh homepage]({{site.url }}/assets/media/redesign-dearbhadh.jpg)
