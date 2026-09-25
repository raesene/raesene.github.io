---
layout: post
title: "Taking a look at Kubernetes Profiling" 
date: 2024-06-18 12:27:00 +0100
comments: false
categories: Kubernetes
---

Debugging facilities can always be interesting for attackers, and in general for security, so I decided to take a look at Kubernetes support for Profiling, and where it could be a risk to cluster security. We'll start with a little bit of background info.

## Golang profiling

Google provides a library called [pprof](https://github.com/google/pprof) that can be embedded in Golang applications to expose profiling information for debugging applications. This allows programs to expose the profiling information via a web server and also provides tools that can visualise and analyse that information.

If you have a program which exposes the pprof information you can then connect to it and start a program to analyse the exposed information. For example if we want to connect to a profiling service at  `http://localhost:8001/debug/pprof/profile`, we'd run

```bash
go tool pprof -http=:8080 http://localhost:8001/debug/pprof/profile
```

and it would give us something like this :-

![pprof web]({{site.url }}/assets/media/pprof-web.png)

From there we can get performance flame graphs and other useful debugging information.

## Kubernetes and profiling

Where this becomes interesting for Kubernetes is that, by default, Kubernetes enables profiling in the API server, scheduler, controller-manager, and Kubelet. Kube-proxy is the odd one out here, I'm not entirely sure why but one guess would be that as the kube-proxy API doesn't support authentication (more on that [here](https://raesene.github.io/blog/2024/06/16/Taking-A-Look-At-The-Kube-Proxy-API/)) it wasn't considered safe to add this functionality to it.

So for each of the APIs that Kubernetes presents you can access the `/debug/pprof/profile` endpoint to get access to this profiling information (and some other functionality discussed in the next section).

Access to this information is not available without credentials and you'll need access to this path via whatever authorization system(s) you've got configured in your clusters. For Kubernetes RBAC, access for kube-apiserver, kube-controller-manager, and kube-scheduler is managed via non-resource endpoints, so typically you'd need to explicitly grant access to that (or be a `cluster-admin` of course).

For the Kubelet, somewhat confusingly, it's gated under the catch-all access of `node/proxy`, so any users with rights to that resource in your cluster will be able to get access to the profiling information.

## Dynamic log level changing

There's also another piece of functionality which is associated with Kubernetes use of profiling, which is the ability to dynamically change the log level of the service remotely. Anyone with access to the profiling endpoints can issue a `PUT` request to change the servers log level. So for example if you've got a `kubectl proxy` running to your cluster on port `8001`, this command would change the API server's log level to `10` which is the DEBUG level.

```bash
curl -X PUT http://127.0.0.1:8001/debug/flags/v -d "10"
```
## Is this really a risk?

So of course the question is, is any of this actually a risk from a security standpoint and, like most things in security, the answer is "it depends". As the profiling information is gated behind authentication and authorization not just anyone can get access, however there are some scenarios where this is a specific risk.

**Managed Kubernetes**. For managed clusters, users who have admin level access to cluster resources are still meant to be restricted from directly accessing the control plane which is managed by the provider. So any managed Kubernetes providers who haven't disabled this on their cluster are at risk of attackers accessing the information. One of the more interesting scenarios where this is relevant, is if the cluster operators can see the Kubernetes API server logs, as it could allow them to exploit [CVE-2020-8561](https://github.com/kubernetes/kubernetes/issues/104720), and start using malicious webhooks to probe the providers network more effectively.

Additionally there's a denial of service risk here for managed providers in that you can do things like start application traces which use control plane node resources. Also it's *possible* that sensitive information could be included in traces, although I've not seen any direct evidence of that in the looking around I've done so far.

For other clusters, there's possibly some risk in that users with `node/proxy` rights can increase the Kubelet's log level and access that information, but that's probably a bit more situational.

## How do I disable profiling in Kubernetes?

If you want to disable profiling in your clusters, and really in production clusters it shouldn't be enabled, you can do it via command line flags or config files. The kube-apiserver, kube-controller-manager, and kube-scheduler all use a parameter called `profiling`, but the Kubelet manages it with `enableDebuggingHandlers`.

## Conclusion

Kubernetes profiling is another interesting part of the API which can be relevant for security. For production clusters, it's an unusual choice to have it enabled by default, and probably something that should just be disabled, unless you explicitly need it. 