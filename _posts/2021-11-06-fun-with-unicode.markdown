---
layout: post
title: "Fun with unicode - messing with output" 
date: 2021-11-06 09:50:39 +0100
comments: false
categories: Kubernetes
---

This week there was some research published from Cambridge university called "[Trojan Codes](https://www.trojansource.codes/)", around the potential risks of RTL unicode characters in source code. Whilst this is very much not a new problem, there have been various pieces of research over the years about the difficulties of handling unicode characters, it seemed like a good cue to look at this kind of issue in the context of Kubernetes. So far I've not found any security issues caused by this, but I found a couple of things which could be of interest, so thought I'd write it down, in case it's useful to anyone.

## Putting unicode into Kubernetes manifests

So, how do you actually put RTL style codes into Kubernetes manifests? Essentially the format is something like this `"\U0000202E"` and putting these codes in a manifest can will change how the information is stored and displayed by Kubernetes. So a concrete example would look like this

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: null
  name: test
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: test
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: "\U0000202Enimda-retsulc\U0000202C"
```

If you create that manifest and then retrieve it, you can see that the `name` in the output shows as cluster-admin with a couple of white spaces in front of it.

![rtl in manifest]({{site.url }}/assets/media/rtl-unicode.png)

## Other possible unicode & output corruption fun

Another area where this kind of thing can cause problems for visual inspection of output is the use of "homoglyph" attacks. It's possible to insert characters which look like an ASCII one but which are not (for example this may look like аdmin but actually the `a` character is a Cryllic character)

Another way I noticed that you can possibly corrupt output is by changing the content of an X.509 certificate, there's more details on that [here](https://github.com/kubernetes/kubectl/issues/888)

## Where might this cause a problem?

So now we've looked at this, the question is, where could this cause problems? The obvious answer here is that, if a cluster administrator is relying on the output of kubectl commands to make security decisions, ways of modifying the output could cause security issues. This is similar to [this issue](https://github.com/kubernetes/kubectl/issues/888) in how CSR information can be represented. 

Another idea might be whether this could be used to bypass external admission controllers. If the admission controller and Kubernetes parse the information in different ways, that might be a possibility.

It's not a problem for the project specifically, but it is something that cluster operators should be aware of.

## Conclusion

The complexities which go with unicode characters are definitely one to be aware of with regards to their possible security impact. There have been multiple cases in the past where security vulnerabilities have occurred due to misinterpretation of UTF-8 input, and I'm sure there will be more in the future :)