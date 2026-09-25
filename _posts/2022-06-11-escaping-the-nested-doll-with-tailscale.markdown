---
layout: post
title: "Escaping the Nested Doll with Tailscale" 
date: 2022-06-11 09:10:39 +0100
comments: false
categories: Kubernetes
---

There are lots of tools which we can use in the container ecosystem to easily create and test applications, but sometimes the networking they create can get a little complex, making it hard to work with and troubleshoot. I came across a scenario recently (for a workshop in Kubecon) where I needed to access a GUI application deployed in a [KinD](https://kind.sigs.k8s.io/) cluster running in an EC2 instance on AWS, from my laptop. The solution I came up with was to use [Tailscale](https://tailscale.com/) and as it seemed like a nice way to solve the problem, I thought it was worth documenting.

## Setting up our Nested Doll

Let's lay out the different networks we're working with to show up the problem. My client machine is on a LAN and has an assigned IP address of `192.168.41.70` (like most home networks I'm using NAT to access the Internet).

I create an EC2 instance, which gets assigned an external IP address of `52.56.82.230` which is in one of AWS' subnets. SSH'ing to that host I get an IP address on my AWS subnet of `172.31.3.70`

```bash
ip addr
ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 06:82:28:cc:5d:6c brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet 172.31.3.70/20 brd 172.31.15.255 scope global dynamic ens5
```

Then I install Docker and KinD on the EC2. Creating a KinD cluster sets up a new Docker container which acts as my node. That container has an IP address of `172.18.0.2`

```bash
docker exec kind-control-plane ip addr
eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth0
```

Then I create a pod in my KinD cluster with the web application that I want to access, which gets an IP address assigned in the pod network range for the cluster of 10.244.0.5

```bash
kubectl get po -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
webserver   1/1     Running   0          13m   10.244.0.5   kind-control-plane   <none>           <none>
```

When all's said and done, it ends up looking a bit like this

![nested doll]({{site.url }}/assets/media/nested-doll.png)

So how do we get easy access to our website? 

## Enter Tailscale

Tailscale is a VPN like product that can be installed on a wide range of devices and essentially creates an overlay network for you that means that devices that you connect can easily access services on any other device in the network. As part of this they have a number of ways of deploying Tailscale to Kubernetes clusters, so you can access your services. In this case probably the easiest to setup is the [subnet router](https://tailscale.com/kb/1185/kubernetes/#subnet-router) where we can essentially give access to any workload in the Kubernetes pod network by deploying a Tailscale pod to act as a router.

Once we follow the Tailscale instructions for creating our Kubernetes Subnet router and [authorize the subnet](https://tailscale.com/kb/1019/subnets/#step-3-enable-subnet-routes-from-the-admin-console) in the admin panel, our networking looks a bit like this

![nested doll with tailscale]({{site.url }}/assets/media/nested-doll-with-tailscale.png)

and if we browse to the pod IP address of our web server (`10.244.0.5`) from our local PC, up pops the deployed application, like magic!

![webapp over tailscale]({{site.url }}/assets/media/tailscale-webapp-access.png)

## Addendum - Getting Cluster DNS

So a question on this post from [Blair Drummond](https://twitter.com/blair_drummond/status/1535952970352930816?s=20&t=9LG336wUxUe36Qjm4BFGwA) was can you get cluster DNS working for this setup? The answer is yes, although it might be a bit fiddly. Let's walk through an example. For this example to work it's important, when following the Tailscale instructions above, that you add a route for the service IP address range in Kubernetes as well as the pod IP address range.

Next, we need to expose our Pod that we created earlier with a service. Services in Kubernetes get DNS names which can be addressed. 

We can do this with something like

```bash
kubectl expose pod webserver --port=80 --target-port=80
```

Once we've done that, we nee to tell Tailscale to use the Kubernetes DNS service for our service domain of `default.svc.cluster.local`. We can do that in the Tailscale DNS section of their admin app.

![webapp over tailscale]({{site.url }}/assets/media/tailscale-cluster-dns.png)

The nameserver IP address we're using here is the Kubernetes DNS service

```bash
kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   13m
```

Once we've got that setup we can reach our webserver that we deployed with a DNS name of `[SERVICENAME].default.svc.cluster.local` and it should work fine :)

## Conclusion

It's pretty easy to end up with complex network setups when playing around in container land, especially once you add the complexities of Kubernetes pod network to the mix. Fortunately there are solutions out there that make it easier to work with all of this and let you get access wherever you need it :)