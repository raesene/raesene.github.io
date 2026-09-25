---
layout: post
title: "The Many IP Addresses of Kubernetes" 
date: 2024-11-01 09:00:00 +0100
comments: false
categories: Kubernetes
---

When getting to grips with Kubernetes one of the more complex concepts to understand is ... all the IP addresses! Even looking at a simple cluster setup, you'll get addresses in multiple different ranges. So this is a quick post to walk through where they're coming from and what they're used for. 

Typically you can see at least three distinct ranges of IP addresses in a Kubernetes cluster, although this can vary depending on the distribution and container networking solution in place. Firstly there is the node network where the container, virtual machines or physical servers running the Kubernetes components are, then there is an overlay network where pods are assigned IP addresses and lastly another network range where Kubernetes services are located. 

We'll start with a standard [kind](https://kind.sigs.k8s.io/) cluster before talking about some other sources of IP address complexity. We'll start by running `kind create cluster` to get it up and running.

Once we've got the cluster started we can see what IP address the node has by running `docker exec -it kind-control-plane ip addr show dev eth0` . The output of that command should look something like this

```bash
13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fc00:f853:ccd:e793::2/64 scope global nodad 
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:2/64 scope link 
       valid_lft forever preferred_lft forever
```

We can see that the address assigned is `172.18.0.2/16`, which is a network controlled by Docker (as we're running our cluster on top of Docker). If you have a Virtual machine or physical server the IP addresses will be in whatever range is assigned to the network(s) the host has.

So far, so simple. Now lets add a workload to our cluster and see what addresses are assigned there. Let's start a webserver workload with `kubectl run webserver --image=nginx`. Once that pod starts we can run `kubectl get pods webserver -o wide` to see what IP address has been assigned to the pod.

```
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
webserver   1/1     Running   0          42s   10.244.0.5   kind-control-plane   <none>           <none>
```

Our pod has an IP address of `10.244.0.5` which is in an entirely different subnet! This IP address is part of the overlay network that most (but not all) Kubernetes distributions use for their workloads. This subnet is generally automatically assigned by the Kubernetes [network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) used in the cluster, so it'll change based on the plugin in use and any specific configuration for that plugin. What's happening here is that our Kubernetes node has created an `veth` interface for our pod and assigned that address to it. We can see the pod IP addresses from the hosts perspective by running `docker exec kind-control-plane ip route` and we can see the IP addresses assigned to the different pods in the cluster, including the IP address we saw from our `get pods` command above.

```
default via 172.18.0.1 dev eth0 
10.244.0.2 dev veth9ee91973 scope host 
10.244.0.3 dev veth1b82cd96 scope host 
10.244.0.4 dev veth38302a10 scope host 
10.244.0.5 dev vethf915cecb scope host 
172.18.0.0/16 dev eth0 proto kernel scope link src 172.18.0.2 
```

Now we've got the node network and the pod network, let's see what happens if we add a Kubernetes [service](https://kubernetes.io/docs/concepts/services-networking/service/) to the mix. We can do this by running `kubectl expose pod webserver --port 8080` which will create a service object for our webserver pod. There are several types of service object, but by default a ClusterIP service will be created, which provides an IP address which is visible inside the cluster, but not outside it. Once our service is created we can look at the IP address by running `kubectl get services webserver`

```
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
webserver   ClusterIP   10.96.198.83   <none>        8080/TCP   97s
```

We can see from the output that the IP address is `10.96.198.83` another IP address range! This range is set by a command line flag on the Kubernetes API server. In the case of our kind cluster, it looks like this `--service-cluster-ip-range=10.96.0.0/16`. 

But from a host perspective, where does this IP address fit in. Well the reality of Kubernetes service objects is that, by default, they're iptables rules created by the `kube-proxy` service on the node. We can see our webserver service by running this command `docker exec kind-control-plane iptables -t nat -L KUBE-SERVICES -v -n --line-numbers`

```
Chain KUBE-SERVICES (2 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        1    60 KUBE-SVC-NPX46M4PTMTKRN6Y  6    --  *      *       0.0.0.0/0            10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
2        0     0 KUBE-SVC-UMJOY2TYQGVV2BKY  6    --  *      *       0.0.0.0/0            10.96.198.83         /* default/webserver cluster IP */ tcp dpt:8080
3        0     0 KUBE-SVC-TCOU7JCQXEZGVUNU  17   --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
4        0     0 KUBE-SVC-ERIFXISQEP7F7OF4  6    --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
5        0     0 KUBE-SVC-JD5MR3NA4I4DYORP  6    --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
6     7757  465K KUBE-NODEPORTS  0    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```


## Conclusion

The goal of this post was just to explore a couple of concepts. Firstly, the variety of IP addresses you're likely to see in a Kubernetes cluster and then how those tie to the operating system level.

