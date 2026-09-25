---
layout: post
title: "Exploring A Basic Kubernetes Network Plugin" 
date: 2024-11-07 13:00:00 +0100
comments: false
categories: Kubernetes
---

In my [last blog](https://raesene.github.io/blog/2024/11/01/The-Many-IP-Addresses-Of-Kubernetes/) I took a look at some of the different IP addresses that get assigned in a standard Kubernetes cluster, but an obvious follow-on question is, how do pods get those IP addresses?, and to answer that question we need to talk about network plugins.

The Kubernetes project took the decision to delegate this part of container networking to external software, in order to make it a more flexible system that can be adapted to different use cases. The way this is done is that the project leverages the [CNI](https://www.cncf.io/projects/container-network-interface-cni/) specification and plugins which comply with that spec. can be used to provide container networking in Kubernetes clusters.

This means that, like many areas of Kubernetes, there's quite a lot of possible complexity and options to consider, and over 20 different network plugins each with their own approach, so let's start with the basics!

## Exploring a basic cluster set-up

We'll make use of [kind](https://kind.sigs.k8s.io/) to provide an initial demonstration cluster, which will give us their default network plugin [kindnetd](https://github.com/kubernetes-sigs/kind/tree/main/images/kindnetd). Kindnetd provide a simple CNI implementation which works well for standard kind clusters. In order to demonstrate how networking works, we'll setup a couple of worker nodes using this config file

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

Then, with that file saved as `kindnet-multi-node.yaml` we can start our test cluster with `kind create cluster --name=kindnet-multi-node --config=kindnet-multi-node.yaml`. Once the cluster's up and running we can take a look at the networking. 

One of the first questions we might have is "how are Kubernetes network plugins configured?". The answer is that any CNI plugins in use have a configuration file in a nominated directory, which is `/etc/cni/net.d` by default. If we look at that directory on our kind nodes we'll see a file called `10-kindnet.conflist` which contains the configuration for the network plugin. Looking at the files in this directory is actually the most reliable way to determine which network plugin(s) are in use as there's no direct record of it at a Kubernetes level.

```json
{
        "cniVersion": "0.3.1",
        "name": "kindnet",
        "plugins": [
        {
                "type": "ptp",
                "ipMasq": false,
                "ipam": {
                        "type": "host-local",
                        "dataDir": "/run/cni-ipam-state",
                        "routes": [
                                { "dst": "0.0.0.0/0" }
                        ],
                        "ranges": [
                                [ { "subnet": "10.244.2.0/24" } ]
                        ]
                }
                ,
                "mtu": 1500
        },
        {
                "type": "portmap",
                "capabilities": {
                        "portMappings": true
                }
        }
        ]
}
```

From this configuration file we can see a bit of how the network plugin works. Firstly we see the `ptp` plugin is used. This plugin is actually one of the default ones that the CNI project maintains. What it does is create a `veth` network interface for each container, which can then be given an IP address. We can also see an `ipam` section which deals with how containers are allocated IP addresses. In this case we can see that a range of `10.244.2.0/24` is assigned to this node, and if we look at the other worker node in the cluster we see it has the `10.244.1.0/24` range,and the control plane node has `10.244.0.0/24`.

So the next question might be "how does the traffic from a pod on one node get to a pod on another node?". This will vary depending on the network plugin you're using but in the case of `kindnet` it's pretty simple. Essentially each node has the entries for the other nodes in its routing table. We can see that by running `ip route` on one of our nodes.

```bash
default via 172.18.0.1 dev eth0 
10.244.0.0/24 via 172.18.0.3 dev eth0 
10.244.2.0/24 via 172.18.0.2 dev eth0 
172.18.0.0/16 dev eth0 proto kernel scope link src 172.18.0.4
```

In this file we can see that the other nodes in the cluster have IP addresses of `172.18.0.3` and `172.18.0.2` respectively, and the container subnets are routed to those nodes.

We can also see how traffic gets to individual pods on that node. First let's create a deployment with 4 replicas `kubectl create deployment webserver --image=nginx --replicas=4`. Once we've got that setup, we can run the `ip route` command again to see what effect that has had.

```bash
default via 172.18.0.1 dev eth0 
10.244.0.0/24 via 172.18.0.2 dev eth0 
10.244.1.2 dev vethc2e31815 scope host 
10.244.1.3 dev veth2621a4f6 scope host 
10.244.2.0/24 via 172.18.0.3 dev eth0 
172.18.0.0/16 dev eth0 proto kernel scope link src 172.18.0.4
```

We can see two new entries in our routing table for the two containers that got started on this worker node, showing how traffic would be sent to the container once it reaches the node.


## Conclusion

This was a quick look at a very simple CNI implementation, and how it all works will vary depending on the network plugin(s) you use. If you're looking for a more in-depth treatment of what we've discussed here, I'd recommend [The Kubernetes Networking Guide](https://www.tkng.io/) which has a lot of information on this topic and others.