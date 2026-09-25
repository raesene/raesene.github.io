---
layout: post
title: "Accessing Cluster IPs from the Outside"
date: 2019-10-03 18:10:39 +0100
comments: false
categories: Kubernetes
---

This is a neat trick which could be useful when troubleshooting Kubernetes services or testing Kubernetes clusters.  This got used in a [TGIK](https://tgik.io) episode a while back and I've been meaning to test it and write it up for a while, as I've not seen many docs on it.

When you run Kubernetes services, generally they are of type "ClusterIP" which means they get assigned a fixed IP address inside the cluster, but this IP address isn't visible externally (it's not designed for external users to contact the service, that's what Ingress resources or NodePort services are for).

However, given the way Kubernetes works, it's possible to access these service IP addresses by the simple expedient of .... Adding a route to them :)  Essentially Kubernetes nodes are Linux routers and kube-proxy will direct traffic for us and doesn't generally care what generated that traffic.

Let's provide a concrete example.  I've got a single node Kubeadm cluster with a single Ethernet interface `ens33` with an IP address of `172.16.198.131`.

The servce IP address range defined on the cluster is the standard `10.96.0.0/12` . If I try to scan a service listening in that range on `10.105.140.70:3000` for example from a host outside the cluster, I'll get told that port is filtered. One point to note here (as suggested by [@TinkerFairy_Net](https://twitter.com/TinkerFairy_Net)) is that filtered from an nmap TCP scan means it didn't receive any response to the request, not that there's some security measure preventing access.

One other point about the command below is that you need the `-Pn` switch to disable nmap's Ping scanning which it usually does to determine if a host is live.  Service IPs aren't real machines, so they won't respond on any port other than the service one.

```
sudo nmap -Pn -sT -n -p3000 10.105.140.70

Starting Nmap 7.60 ( https://nmap.org ) at 2019-10-03 19:51 BST
Nmap scan report for 10.105.140.70
Host is up.

PORT     STATE    SERVICE
3000/tcp filtered ppp

Nmap done: 1 IP address (1 host up) scanned in 2.04 seconds
```

So we can't get to that port on that IP address. Now lets try adding a route to the service IP address network via the cluster node's main IP address. 

```
sudo ip route add 10.96.0.0/12 via 172.16.198.131
```

This change essentially tells our client machine that there's another router on the network and that for traffic in the range `10.96.0.0/12` it should send the packets to our Kubernetes cluster node for onwards transmission.

Now if we retry the same command

```
sudo nmap -Pn -sT -n -p3000 10.105.140.70

Starting Nmap 7.60 ( https://nmap.org ) at 2019-10-03 19:53 BST
Nmap scan report for 10.105.140.70
Host is up (0.00074s latency).

PORT     STATE SERVICE
3000/tcp open  ppp

Nmap done: 1 IP address (1 host up) scanned in 0.03 seconds
```

Our port is open :)

So we can interrogate any service in that network from outside the cluster (and without any rights to the cluster!) without restriction.

From a security point, the important thing to take away here is, don't assume that services are protected because they're not hooked into an ingress, if an attacker can route traffic to the cluster, they can just do this to see those IP addresses.