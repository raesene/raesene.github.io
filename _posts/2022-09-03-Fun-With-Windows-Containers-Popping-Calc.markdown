---
layout: post
title: "Fun with Windows Containers - Popping Calc" 
date: 2022-09-03 17:00:00 +0100
comments: false
categories: Kubernetes
---

Windows containers don't get quite the use of their Linux brethren, but they're an interesting topic and one that's seeing more adopting as enterprises move to Containerization. Whilst, from a Docker/Kubernetes perspective, they look relatively similar to Linux containers, the underlying isolation mechanisms are entirely different. A new development in this is the provision of "host process" containers, so I thought it would be fun to take a look at what's possible with them, but first some background...

# Background

## Isolation Options

There are two isolation options available to users of Windows containers, Process-isolated Windows Server containers and Hyper-V isolated Windows Server containers. [Microsoft's documentation](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container) goes into a lot of detail about the two, but a key difference is that process containers use the same kernel (like Linux containers) whereas Hyper-V isolation provides a full operating system kernel per container and uses hypervisor isolation.

Importantly to quote [Microsoft's documentation on container security](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-security#container-security-servicing-criteria) `Only hypervisor-isolated containers provide a security boundary, and process-isolated containers do not.` 

With that said whilst Microsoft don't consider process containers to provide a security boundary, they have decided that some container breakouts should be considered privilege escalation and have fixed [issues related to this](https://googleprojectzero.blogspot.com/2021/04/who-contains-containers.html) in the past.

## Windows Containers and Kubernetes

It's possible to use Windows containers as part of a Kubernetes cluster, although you still need Linux based control plane servers. When running Windows containers in a Kubernetes cluster, [Hyper-V isolation is not supported](https://kubernetes.io/docs/concepts/windows/intro/#windows-containers-in-kubernetes), so containers will use process based isolation

# Host Process Containers

This is a new feature in Kubernetes which is currently in beta and scheduled to reach GA in Kubernetes 1.26, although it should work on any cluster at 1.23 or higher out of the box. The idea of this feature is to replicate the [privileged container feature in Linux](https://github.com/kubernetes/enhancements/issues/1981), which essentially removes security isolation layers between the container and the host. This is intended to be used by system level components that need additional access to the underlying host, for example CNI providers.

## Getting a container to pop calc :)

With the background out of the way and 2-node cluster set-up (Calico have some [good documentation](https://projectcalico.docs.tigera.io/getting-started/windows-calico/quickstart) on the setup process), the question obviously is "can we get a container workload to pop calc on a cluster node!".

The way I approached this was to use [psexec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) to execute a process on my cluster node. We just need a manifest which references an image containing that program and we should be able to execute programs there.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wintest
spec:
  containers:
  - name: test
    image: raesene/windows-powertools
    command: ["$env:CONTAINER_SANDBOX_MOUNT_POINT\\pstools\\psexec.exe", "-i", "1", "-accepteula", "c:\\windows\\system32\\calc.exe"]
  securityContext:
    windowsOptions:
      hostProcess: true
      runAsUserName: "NT AUTHORITY\\SYSTEM"
  hostNetwork: true
  nodeSelector:
    "kubernetes.io/os": windows
```

A couple of interesting points from the manifest. I found that passing `-i 1` was needed to get the program to execute in the interactive session running on the server (you can specify a higher integer to get other interactive sessions on the server). Also you can specify `NT AUTHORITY\\SYSTEM` as the running user which gives you an idea of the privileges available to a host process container. We also use a `nodeSelector` field to ensure that this container will run on a Windows node in our cluster.

If you wanted to replicate this on every Windows node in a cluster, you'd just use a daemonset instead of a pod.

To save people from the hassle of setting up a cluster to test this, here's a quick video

<iframe width="560" height="315" src="https://www.youtube.com/embed/KbcG2tWsmt0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Getting shell on the host

If you're looking for something a bit more interactive to look around the node(s), a manifest like this will start up and pause long enough for you to connect to the host

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: winshell
spec:
  containers:
  - name: winshell
    image: raesene/windows-powertools
    command:
    - powershell.exe
    - -command
    - "start-sleep -seconds 600"
  securityContext:
    windowsOptions:
      hostProcess: true
      runAsUserName: "NT AUTHORITY\\SYSTEM"
  hostNetwork: true
  nodeSelector:
    "kubernetes.io/os": windows
```

You can then get a nice interactive shell using kubectl

```bash
kubectl exec -it winshell -- cmd.exe
```

# Conclusion

If you're planning to run Windows containers under Kubernetes, it's important to make sure you're restricting this new feature appropriately. Admission Control solutions like [Kyverno](https://kyverno.io/) or [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper) can be used to restrict this.

Also this is an interesting area to dig into more, as I'd guess we'll see its usage increase over time.

