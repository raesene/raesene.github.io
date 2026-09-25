---
layout: post
title: "Getting into a bind with Kubernetes" 
date: 2021-01-16 17:00:39 +0100
comments: false
categories: Kubernetes
---

Following on from [looking at the escalate verb in Kubernetes RBAC](https://raesene.github.io/blog/2020/12/12/Escalating_Away/), I thought it would be worth looking at another one of the unusual verbs you can see in Kubernetes RBAC, bind.

These two, along with the impersonate verb are operations that are available on some RBAC objects in the Kubernetes API. They're quite important if you're looking at designing or auditing Kubernetes rights as they can allow for privilege escalation, so know which principals have access to them is important.

The bind verb can be applied to roles or clusterroles and allows a principal to bypass a general restriction on (cluster)role binding creation, which stops users who can create role bindings from escalating their privileges by binding to high privilege roles like cluster admin. This restriction is described in the Kubernetes documentation [here](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-binding-creation-or-update)

The way that this works is that, without using bind, a user cannot create a rolebinding or clusterrolebinding to a role or cluster role which has rights that the user themselves does not currently have. So for example if a user with the rights to create rolebindings tries to bind to cluster-admin , this won't work unless that user already has cluster-admin rights.

Bind is made available to allow for this restriction to be bypassed.

## Practical Example

Let's show a practical example of this. First up we'll create a couple of service accounts to act as our user for this experiment and we'll call them `rbac-binder` and `not-rbac-binder`

Now we'll create a couple of cluster roles. Both have `create` on `clusterrolebindings` but one will also have `bind` on `clusterroles`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rbac-binder
rules:
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterroles
  verbs:
  - bind
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  verbs:
  - create
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: not-rbac-binder
rules:
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  verbs:
  - create
```

Then we'll just bind those two clusterroles to their respective service accounts.

So at this point if we look at our users rights with ` kubectl --as=system:serviceaccount:default:rbac-binder auth can-i --list` we can see that the rbac-binder service account has the following rights

```bash
Resources                                       Non-Resource URLs   Resource Names   Verbs
clusterroles.rbac.authorization.k8s.io          []                  []               [bind]
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
clusterrolebindings.rbac.authorization.k8s.io   []                  []               [create]
                                                [/api/*]            []               [get]
                                                [/api]              []               [get]
                                                [/apis/*]           []               [get]
                                                [/apis]             []               [get]
                                                [/healthz]          []               [get]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/livez]            []               [get]
                                                [/openapi/*]        []               [get]
                                                [/openapi]          []               [get]
                                                [/readyz]           []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
                                                [/version]          []               [get]
```

and the not-rbac-binder service account has these rights

```bash
 kubectl --as=system:serviceaccount:default:not-rbac-binder auth can-i --list
Resources                                       Non-Resource URLs   Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
clusterrolebindings.rbac.authorization.k8s.io   []                  []               [create]
                                                [/api/*]            []               [get]
                                                [/api]              []               [get]
                                                [/apis/*]           []               [get]
                                                [/apis]             []               [get]
                                                [/healthz]          []               [get]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/livez]            []               [get]
                                                [/openapi/*]        []               [get]
                                                [/openapi]          []               [get]
                                                [/readyz]           []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
                                                [/version]          []               [get]
```

Now if we try to use `not-rbac-binder` to create a clusterrolebinding to cluster-admin, it will fail with the following error message (remember here that we do have create clusterrolebinding rights).

```bash
kubectl --as=system:serviceaccount:default:not-rbac-binder create clusterrolebinding rbac-clusteradmin --clusterrole=cluster-admin --serviceaccount=namespace:not-rbac-binder
error: failed to create clusterrolebinding: clusterrolebindings.rbac.authorization.k8s.io "rbac-clusteradmin" is forbidden: user "system:serviceaccount:default:not-rbac-binder" (groups=["system:serviceaccounts" "system:serviceaccounts:default" "system:authenticated"]) is attempting to grant RBAC permissions not currently held:
{APIGroups:["*"], Resources:["*"], Verbs:["*"]}
{NonResourceURLs:["*"], Verbs:["*"]}
```

Then if we try the same operation with `rbac-binder` it works!

```bash
kubectl --as=system:serviceaccount:default:rbac-binder create clusterrolebinding rbac-clusteradmin --clusterrole=cluster-admin --serviceaccount=namespace:rbac-binder
clusterrolebinding.rbac.authorization.k8s.io/rbac-clusteradmin created
```


## Conclusion

The bind verb on RBAC is a useful piece of Kubernetes RBAC configuration but it's also a lesser known are of RBAC and one which not all tools might properly understand. As it can allow for privilege escalation, it's an important thing to check for when creating or auditing roles and clusterroles.