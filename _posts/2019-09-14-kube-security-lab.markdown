---
layout: post
title: "Kubernetes Security Lab with Kind and Ansible"
date: 2019-09-14 17:10:39 +0100
comments: false
categories: Kubernetes
---

Being able to practice exploits and attacks is always useful for security testers, whether it's working out whether a tool is working properly, or fine-tuning the syntax for a command in a predictable environment, it's a very handy technique.  One factor that can slow this down is having to rely on external resources, like Virtual Machines or cloud based resources, for running our tests.  Ideally we should be able to run everything locally on a single machine.

In the past I've looked at using [kind](https://kind.sigs.k8s.io) for this (with [kind of insecure](https://raesene.github.io/blog/2019/03/04/kind-of-insecure-test-clusters/)).  This works pretty well, but there are some limitations on what we can do in terms of setting up vulnerable environments with just kind on it's own.

Adding a configuration management tool to the mix can let us easily create more complex test environments.  Enter [Ansible](https://www.ansible.com/) which works pretty well for this application. It doesn't require any server infrastructure, which is good for this kind of setup, and it's possible to define a Docker container as the host for applying the actions to via a playbook.

## Kube Security Lab

So I've started off the process of creating a set of vulnerable clusters as Ansible playbooks and put it [here](https://github.com/raesene/kube_security_lab)

The idea is that the `client-machine.yml` playbook can be used to spin up a container with client tools installed (it's just an instance of [this image](https://hub.docker.com/r/raesene/alpine-containertools) at the moment) and then bring up one or more of the vulnerable clusters as playbooks, practice attacking that configuration and then easily remove both the cluster and client container.

In general you can spin up the client machine and a sample cluster, then port-scan the target cluster to see what's exposed and start attacking things!

There's a starter set of playbooks up now, and I'll plan to expand this as I get more ideas.  Also there should be walkthroughs for each of the clusters, in case people want the cheat sheet version :)

