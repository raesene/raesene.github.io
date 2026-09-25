---
layout: post
title: "When is read-only not read-only?" 
date: 2024-11-11 13:00:00 +0100
comments: false
categories: Kubernetes
---

Bit of a digression from the network series today, to discuss something I just saw in passing which is an interesting example of a possible sharp corner/foot gun in Kubernetes RBAC.

Generally speaking for REST style APIs `GET` requests are read-only, so shouldn't change the state of resources or execute commands. As such you might think that giving a user the following rights in Kubernetes would essentially just be giving them read-only access to pod information in the default namespace.


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: 
    - "pods"
    - "pods/log"
    - "pods/status"
    - "pods/exec"
    - "pods/attach"
    - "pods/portforward"
  verbs: ["get", "list", "watch"]
```

However due to the details of how Websockets works with Kubernetes, this access *can* allow for users to run `kubectl exec` commands in pods and get command execution rights in that namespace! There's information on the origins of this in [this Github issue](https://github.com/kubernetes/kubernetes/issues/78741) but it's essentially down to how websockets works.

What's possibly more interesting is that, while this behaviour has been in place for a while you might not have noticed it, as the default in Kubernetes was to use [SPDY](https://en.wikipedia.org/wiki/SPDY) for `exec` commands instead of websockets, until Kubernetes version 1.31. So if a user with `GET` rights on `pods/exec` tried to use `kubectl exec` in 1.29 you'd get an error like this

```
Error from server (Forbidden): pods "test" is forbidden: User "bob" cannot create resource "pods/exec" in API group "" in the namespace "default"
```

but if a user with the exact same rights, tried the same command in Kubernetes 1.31 it works!

```
kubectl --kubeconfig bob.config exec -it test -- /bin/bash
bash-5.1# exit
exit
```

It's worth noting that, whilst it's easier to do now, using websockets with these rights has been possible for a long time using tools like [kubectl-execws](https://github.com/jpts/kubectl-execws) from [jpts](https://hachyderm.io/@jpts).

## Conclusion

Kubernetes RBAC has some tricky areas where the behaviour you get might not be exactly what you expect, and sometimes as in this case, those unexpected behaviours are not very apparent!