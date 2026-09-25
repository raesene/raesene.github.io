---
layout: post
title: "Fun with SSRF - Turning the Kubernetes API Server into a port scanner" 
date: 2023-01-02 09:27:00 +0100
comments: false
categories: Kubernetes
---

I thought I'd start the new year with something a little fun that I've been looking at over the break (well for a certain definition of the word 'fun' :) ). Kubernetes has quite a rich API and in the various objects that you can create, some of them have URL or Service fields which, when used, cause the Kubernetes API server itself to make network requests (generally over HTTPS). Knowing this, it feels a bit like a Server-Side Request Forgery (SSRF) attack, so I wondered how possible it would be to implement something that can be used to scan for open ports on a target host from the Kubernetes API server.

An important point to note here, is that this is all standard Kubernetes functionality, no 0-days or vulnerabilities are involved. To carry out this process you need to be able to create some high-privileged objects in the cluster, so in most cases there's no privilege escalation involved.

One slight exception to this, is that if you're using Managed Kubernetes (AKS, EKS, GKE etc) you can use this to port scan some parts of the CSPs network, but this is just information disclosure and I'm sure their security architectures are robust enough that simple port scanning presents no real threat.

## Using Validating Admission Webhooks

The first object I thought of using for this is Validating Admission Webhooks, as they take either a service or URL as part of their specification, then when they receive a request for an in-scope object the Kubernetes API server passes the request to that URL, so it fits our profile.

If we want to implement this technique, the next step is to work out how to trigger it and also do so in a relatively safe way, to avoid disrupting the overall operation of the cluster, while we're port scanning. To do this we can create a validating admission webhook configuration that only looks at requests in a single namespace. If we create a dedicated namespace for the scanning, then we can ensure that the webhook only looks at requests in that namespace, and we can also delete the namespace when we're done to clean up.

Once we have our namespace and webhook, we just try to create a pod in that namespace, expecting it to fail, and then record the error message returned.

So, a simplified version of our flow should look a little like this :

![Sequence diagram for SSRF]({{site.url }}/assets/media/kubernetes_vaw_ssrf.png)

Trying this out manually, one thing that I noticed which is very handy for our purposes is that the Kubernetes API provides a verbose error message depending on what sort of error it encountered. This lets us differentiate between "no response", "port closed" and "port open", with some added details like "this port was open but didn't speak HTTP".

## Automating the process

Whilst it's perfectly possible to do this manually, it's pretty time-consuming as you need do have a set of steps like this

- Edit the template webhook manifest with the target host and port.
- Check if the namespace exists, if not create it.
- Check if the webhook exists, if it does delete it.
- Create the new webhook.
- Create a pod in the namespace.
- Check the error message returned by the API server when it tries and fails to call the target admission webhook.
  - Return this to the user after interpreting the error for what it indicates.
- Delete the webhook and the namespace.

So I did what any good lazy person would do, and wrote some code to do it for me :)

With the reminder of "don't run this on production clusters!", the PoC code is available [here](https://github.com/raesene/k8s_ssrf_portscanner). You can use it to scan host/port combinations from the perspective of a Kubernetes API server. The code will try to interpret the error message that comes back and tell you if the port is unreachable/closed/open.

## Demo

Here's a quick demonstration of how this works. In the video I've got an AKS cluster up and running and I'll use the SSRF port scanner to hit a URL I control, so we can see the request (caddy.pwndland.uk).

In the logs you can see the source IP of 40.88.207.52 and User-agent of "kube-apiserver-admission" showing that it's the API server making the request.

<iframe width="560" height="315" src="https://www.youtube.com/embed/y9QBDlmk0jA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Conclusion

This post just shows how it's possible to leverage existing functionality on Kubernetes to perform scans from the perspective of the API server using validating admission webhooks, an interesting side-effect of how the API server is designed. There are other objects you could use for this I'm sure :)
