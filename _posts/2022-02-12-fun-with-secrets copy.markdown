---
layout: post
title: "Fun with secrets - Where did they go?" 
date: 2022-02-12 15:50:39 +0100
comments: false
categories: Kubernetes
---

Before I get into this post a quick note that there is no dramatic payoff here, it's just playing around with something that surprised me in Kubernetes, to understand a bit about what's going on.

Kubernetes clusters make use of secrets for a variety of purposes and (for the time being) one of the main ones is to provide credentials to service accounts, so that the workloads running in a cluster can authenticate to the API server. The reason I say "for the time being" is that this feature is being replaced by [Bound Service Account Token Volumes](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume), but that's a matter for another post.

The way it usually works is that, when you create a service account, Kubernetes will automatically create a secret to go along with it, and then when you specify that service account is to be used in your workload, Kubernetes will automatically mount the secret into the workload for you.

Whilst this is usually automatic, when I was chatting to [Christophe](https://twitter.com/christophetd) the other day, he mentioned that you can manually create secrets and add an annotation to them specifying a service account name. When you do that, Kubernetes will automatically populate the secret with a service account token for that service account.

When playing around with this and creating new secrets, I noticed something odd. If you specify a service account name that doesn't exist in your namespace when creating your secret, while kubectl tells you the creation happens ok, the secret never actually appears!. So as an example, if you get a secret like this

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-robot
type: kubernetes.io/service-account-token
```

save it in a file called, `build-robot-secre.yaml` and do `kubectl create -f build-robot-secret.yaml` you get

```shell
kubectl create -f base-secret.yaml
secret/build-robot-secret created
```

but then if you do `kubectl get secrets`

```shell
kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-nxrbt   kubernetes.io/service-account-token   3      40m
```

it's nowhere to be seen!

## Working out what's happening

So we have a mystery, what is causing our secret (which is valid kubernetes YAML) to seem to be created but apparently vanish after creation. The first thought I had was, maybe it's being created but not displayed. So to check that, we can look at the contents of the cluster's etcd database. It's the canonical store of information for a cluster, so generally a good place to look.

To do this, I set-up a cluster with etcd authentication turned off (to make things easier to check), using the etcd-noauth playbook from [kube-security-lab](https://github.com/raesene/kube_security_lab/). with that set-up we can rerun our secret creation loop and see what does or doesn't show up in etcd, using etcdctl.  The layout of the database is pretty straightforward, so we just ask it to show us the contents of /registry/secrets/default, which is all the secrets in the default namespace.

```shell
 etcdctl --insecure-transport=false --insecure-skip-tls-verify --endpoints=172.18.0.3:2379 get /registry/secrets/default --prefix --keys-only
/registry/secrets/default/default-token-nxrbt
```

Looking at the output, we can see it's just our default service account token. So now we know that the secret isn't persisted. Next question is, what happens if we change an existing secret which points to a valid service account, and make it point to an invalid one.

We can create a valid one like this

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: extra-default-secret
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
```

and if we do `kubectl get secrets` once we've created it, we can see it's there

```shell
❯ kubectl get secrets
NAME                   TYPE                                  DATA   AGE
default-token-nxrbt    kubernetes.io/service-account-token   3      5h20m
extra-default-secret   kubernetes.io/service-account-token   3      4s
```

ok so now if we do `kubectl edit secret extra-default-secret` and change the `service-account.name` field to `test` what'll happen... The answer turns out to be that the edit succeeds and the secret vanishes! Checking etcd as above confirms that our secret has gone from the datastore too.

## Why does that happen?

Knowing a bit about Kubernetes I've got a fair idea of why this happens, I'd expect it to be done by one of the [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) that Kubernetes use to manage cluster state. There are a wide range of controllers which run as part of the controller-manager component and what they do (roughly) is run a loop watching the state of a certain class of object looking at the desired state. They then make changes to the configuration of the cluster to ensure it stays in that desired state.

Looking at the docs for [Kubernetes service accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#token-controller) we see this

```
TokenController runs as part of kube-controller-manager. It acts asynchronously. It:

    watches ServiceAccount creation and creates a corresponding ServiceAccount token Secret to allow API access.
    watches ServiceAccount deletion and deletes all corresponding ServiceAccount token Secrets.
    watches ServiceAccount token Secret addition, and ensures the referenced ServiceAccount exists, and adds a token to the Secret if needed.
    watches Secret deletion and removes a reference from the corresponding ServiceAccount if needed.
```

So it seems likely, although the docs don't mention it deleting secrets that don't match to an existing service account, that this component will be the one deleting our secrets, but how do we check?

Well Kubernetes auditing should be able to help us here, as all the controllers in Kubernetes send their requests via the API server, so we can look at what happens when we try to create our `build-robot-secret`. 

Running the `kubectl create` from earlier on that cluster confirms the hypothesis. There's three events which happen.

1. A create event from the kubernetes-admin user (which is the name of the default first user in a kubeadm cluster)
2. A get event from the tokens-controller. Interestingly here the "user" is `system:kube-controller-manager` but the user agent indicates that it's the tokens-controller.
3. A delete event from the tokens-controller that looks like this

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "RequestResponse",
  "auditID": "ef0f48eb-a606-4f50-afcd-517268a5cd2e",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/secrets/build-robot-secret",
  "verb": "delete",
  "user": {
    "username": "system:kube-controller-manager",
    "groups": [
      "system:authenticated"
    ]
  },
  "sourceIPs": [
    "172.18.0.3"
  ],
  "userAgent": "kube-controller-manager/v1.21.1 (linux/amd64) kubernetes/5e58841/tokens-controller",
  "objectRef": {
    "resource": "secrets",
    "namespace": "default",
    "name": "build-robot-secret",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "metadata": {},
    "status": "Success",
    "code": 200
  },
  "requestObject": {
    "kind": "DeleteOptions",
    "apiVersion": "v1",
    "preconditions": {
      "uid": "16c9f476-34fe-48a0-b623-100f7de93d0f"
    }
  },
```

## Conclusion

Like I mentioned at the top of this post, nothing earthshaking here, but it's kind of interesting to dig into how Kubernetes controllers which keep the cluster in a desired state are constantly watching and changing resources.

In terms of impact, outside of a niche scenario where someone was allowed to edit secrets but not delete them, there's probably no major impact here, but it might help someone understand why a secret that looked like it got created ok, was no longer present in their cluster!
