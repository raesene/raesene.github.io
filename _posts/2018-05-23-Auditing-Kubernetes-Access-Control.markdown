---
layout: post
title: "Auditing Kubernetes Access Control"
date: 2018-05-23 20:45:39 +0100
comments: false
categories: Kubernetes
---

A common task in any security review, is auditing user access control, as excessive numbers of privileged users are a common theme, and privileged access a common point of attack.

However when it comes to completing this kind of review on a Kubernetes cluster, you'll likely find it not as straight-forward as expected, due to k8s' rather hands-off approach to identity management.

Similarly to areas such as Networking and Storage, k8s has taken the decision to delegate the matter of user identity management to largely external systems.  There is no standard user database in a cluster, and where you can find the information to review privileged access will largely depend on the configured authentication mechanisms.

## What information is available?

Assuming that you're using RBAC ('cause if you're not it's likely the answer to "who has cluster-admin rights?" is "All your authenticated users") the information available is from a couple of different sources. `clusterroles` and `clusterrolebindings` provide information on cluster level privileges and `roles` and `rolebindings` serve the same purpose at the namespace level.

What you can get from the cluster is a list of the subjects that have a given role.  These can be either users, service accounts, or groups.  The main issue, in terms of determining the overall access to the cluster, comes from the inclusion of groups in that list of subjects, as group membership isn't recorded anywhere within the cluster, it's defined by the identity provider.

To make this possbly a little clearer, lets take a worked example.

The cluster role `cluster-admin` is obviously pretty key as it has complete rights to the entire Kubernetes cluster.  On a standard kubeadm 1.9 cluster if you run the command

`kubectl get clusterrolebinding cluster-admin -o yaml`  the output will look something like

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2018-01-10T21:03:11Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "94"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/cluster-admin
  uid: aa8f62e2-f649-11e7-8092-000c290b2418
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters
```

The key point is the subjects section at the end where we can see that all members of the `system:masters` group have the `cluster-admin` role.  The membership of that group isn't defined anywhere in your cluster.

It's worth noting also that whilst tying up the identity of people with rights to your cluster is easier with Users and Service accounts than it is with groups, Kubernetes still relies on external systems to warrant these identities, so just 'cause the cluster role binding says "User fred has a binding to the cluster-admin role", it doesn't actually have any way (within the cluster) of asserting who fred is.

## So how do I audit k8s user rights?

Basically you'll need to find out what authentication mechanisms are supported by the cluster, and then for each one, determine how group membership is defined and see if you can assemble a list of users that way.

For example, lets say the cluster is using client certificate authentication (a common option in many clusters).  As per  [the Kubernetes docs](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certs) group memberships are defined in the CSR, so to review membership, you'll need to grep. through all the CSRs (assuming there is a record of them) and pull out the group memberships that way.

## Conclusion

Nothing in this post will come as any surprise to people who've been following Kubernetes security for a while now.  However as it grows in popularity and more large companies start rolling it out, issues like understanding who has rights to what will start becoming more important.