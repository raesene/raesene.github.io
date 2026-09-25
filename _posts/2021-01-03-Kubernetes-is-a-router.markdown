---
layout: post
title: "Kubernetes is a router" 
date: 2021-01-03 17:00:39 +0100
comments: false
categories: Kubernetes
---

One of the interesting areas of Kubernetes to explore is how it handles networking, and this is a quick post looking at one assumption which can be made about Kubernetes networking and how it doesn't always hold. Whilst Kubernetes can assign pod IP addresses on the same LAN as the host VM, it's more normal for the CNI to hand out IP addresses from a separate range, and then implement some kind of overlay networking between the cluster nodes to get traffic from pods on one node, to pods on other nodes. As part of this duty, Kubernetes nodes will often act as routers, so if you can get traffic to the node they'll happily forward it on.

This could leave you with the idea that pods aren't accessible from elsewhere on the LAN that the cluster lives on, which could in turn provide a false sense of security about their accessibility, so it's important to know that whilst pods aren't usually accessible, that's not a security feature.

To demonstrate this I setup a single node Kubeadm cluster and a client machine on the same LAN. In this setup the cluster node is at `192.168.197.134` and the client at `192.168.197.135`.

On the cluster node if I do `kubectl get po -A -o wide` we can see the pod IP addresses for the pods running in the cluster :-

```bash
NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE   IP                NODE      
calico-system     calico-kube-controllers-546d44f5b7-gchk4   1/1     Running   1          16h   10.8.203.8        kubeadm120
calico-system     calico-node-475ff                          1/1     Running   1          16h   192.168.197.134   kubeadm120
calico-system     calico-typha-54879db669-6m4pr              1/1     Running   1          16h   192.168.197.134   kubeadm120
default           testweb                                    1/1     Running   1          16h   10.8.203.7        kubeadm120
kube-system       coredns-74ff55c5b-pp2kt                    1/1     Running   1          16h   10.8.203.6        kubeadm120
kube-system       coredns-74ff55c5b-x8fzx                    1/1     Running   1          16h   10.8.203.5        kubeadm120
kube-system       etcd-kubeadm120                            1/1     Running   1          16h   192.168.197.134   kubeadm120
kube-system       kube-apiserver-kubeadm120                  1/1     Running   1          16h   192.168.197.134   kubeadm120
kube-system       kube-controller-manager-kubeadm120         1/1     Running   1          16h   192.168.197.134   kubeadm120
kube-system       kube-proxy-kkb2v                           1/1     Running   1          16h   192.168.197.134   kubeadm120
kube-system       kube-scheduler-kubeadm120                  1/1     Running   1          16h   192.168.197.134   kubeadm120
tigera-operator   tigera-operator-657cc89589-624rj           1/1     Running   2          16h   192.168.197.134   kubeadm120
```

Whilst some of them (the ones using host networking) are on the `192.168.197.0/24` network, some of the others are on the pod network of `10.8.0.0/16` which was configured when I setup the cluster. So, from my client machine, I can't immediately address services on the pod network. If I try to access the nginx web server on the "testweb" pod, it won't work, as my client doesn't know how to route traffic to it.

However, like the post title says, Kubernetes is a router, so if I can add a route on my client, telling it that the `10.8.0.0/16` network can be reached via the Kubernetes cluster node, I can get traffic to it!

This is easily enough done, just add a route like this `sudo route add -net 10.8.0.0 netmask 255.255.0.0 gw 192.168.197.134` and then bingo, `curl http://10.8.203.7` will return our nginx container's home page.

## Why does that work?

So we can then look a bit into why this works. The first part is routing. If we look at the routing table on our cluster node we can see routes for our pods. `route -n` shows :-

```bash
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.197.2   0.0.0.0         UG    100    0        0 ens33
10.8.203.5      0.0.0.0         255.255.255.255 UH    0      0        0 cali8c667e24141
10.8.203.6      0.0.0.0         255.255.255.255 UH    0      0        0 calib967dfbd495
10.8.203.7      0.0.0.0         255.255.255.255 UH    0      0        0 calib9199d171be
10.8.203.8      0.0.0.0         255.255.255.255 UH    0      0        0 calie65c0ecd96a
10.8.203.9      0.0.0.0         255.255.255.255 UH    0      0        0 cali68caf03a5f4
10.8.203.10     0.0.0.0         255.255.255.255 UH    0      0        0 cali78a7a37ae9d
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.197.0   0.0.0.0         255.255.255.0   U     0      0        0 ens33
192.168.197.2   0.0.0.0         255.255.255.255 UH    100    0        0 ens33
```

So we know the node has a route for the pods which is local as there's no "G" flag on it. The other part of the equation is packet forwarding. Checking the standard Linux sysctl for this with `sysctl net.ipv4.ip_forward` we can see that packet fowarding is enabled

```bash
net.ipv4.ip_forward = 1
```

So traffic sent to the kubernetes node will be forwarded, based on its routing table, and will get to the pod ok.

## Conclusion

This is a relatively minor point, but one worth remembering, just because you can't get access to pod IP addresses from outside the cluster by default, it's not a security barrier you should be relying on. In general services running in pods on the cluster should be secured, just like any other service running in your network as they are likely to be accessible in one way or another. If you do want to restrict access to pods running in a cluster, using [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) will be the way to go.