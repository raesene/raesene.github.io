---
layout: post
title: "Exploration in Docker Bridging"
date: 2016-02-07 15:45:39 +0100
comments: false
categories: Docker
---


One of the things I've been interested to look at with docker is the network setup. By default when you bring up a docker container you get a network interface with a private IP address which can communicate with other containers on that network and can make outbound connections to the wider world, but isn't visible to the wider network.

Docker does this by attaching the container to a bridge on the host and setting up iptables NAT rules to allow traffic to flow from this bridged network to the wider external network.  There's a couple of downsides to this approach.  Firstly you need to explicitly specify inbound ports at container launch time if you want to have server services receive connections.  When you do this, docker sets up port-forwarding to allow that traffic into the container.  

The problem with this approach is that if you're running a service that sets ports up on the fly or which may change port during it's runtime (e.g. metasploit) this is a bit on the inflexible side.

The other downside is the use of NAT, which can cause issues with some networking tools that make a large number of network connections (e.g. nmap).

One option to address this is to use the docker option ```-net=host``` which essentially has the container run with the hosts network settings.  However one slight problem there is.... it doesn't work with user namespaces, which I'm keen on having enabled. 

So ideally I'd like another option which provides no-NAT access to a container, ideally with an IP address on the host network subnet, with user namespaces enabled.  This is kind of similar to the VMWare Workstation/fusion setup with bridged networking.

I had quite a dig around in the official docker documentation to see if this kind of setup is possible using in-built functionality and I didn't see a way of doing this, so a little manual work is required. However I did find some interesting references to working out this kind of problem notably this [StackOverflow post](https://stackoverflow.com/questions/25529386/how-can-i-set-a-static-ip-address-in-a-docker-container) and [this](http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/) slightly older post on Docker networking.

First in one shell we need to start our target container.  We use the option ```--net=none``` to setup the container without having docker do it's usual network setup routines.

{% highlight shell %}
docker run -it --net=none --name=nettest ubuntu:14.04 /bin/bash
{% endhighlight %}

Next we need to create our virtual interface as below, I've called it ```virtual0``` , in my case ```eno16777736``` is the name of my host ethernet interface

{% highlight shell %}
sudo ip link add virtual0 link eno16777736 type macvlan mode bridge
{% endhighlight %}

After we're done that we need to add the virtual0 interface to the container, so we need to know the pid, which we can get from docker inspect.

{% highlight shell %}
docker inspect --format '{ { .State.Pid }}' nettest
{% endhighlight %}

Then we create a link in /var/run/netns and set the netns of the virtual0 interface to be the same as our container.  In the examples below 3463 is the pid returned by the docker inspect command above.

{% highlight shell %}
sudo mkdir -p /var/run/netns
sudo ln -s /proc/3463/ns/net /var/run/netns/3463
sudo ip link set virtual0 netns 3463
{% endhighlight %}

We can then bring up the interface and apply an IP address.  Doing this inside the container doesn't work from a permissions standpoint so it can be done with the ```ip netns exec``` command from the host

{% highlight shell %}
sudo ip netns exec 3463 ip link set virtual0 up
{% endhighlight %}

and we can then assign a DHCP address from the host network DHCP server to the container

{% highlight shell %}
sudo ip netns exec 3463 dhclient virtual0
{% endhighlight %}

Obviously doing all this manually is likely to be a bit much trouble for most use-cases, but I think with a bit of automation it could come in handy for some tasks which need a bit more flexibility in networking than the default docker configuration.

**Edit:** I did find [this project](https://github.com/gopher-net/macvlan-docker-plugin) which looks to address the same problem, but at the moment it doesn't seem to work for me...

**Edit2:** Well that was quick. The problems I had with the macvlan docker plugin got fixed (thanks to @nerdalert), so for a faster/less manual solution to this problem I'd recommend having a look at the link in the first edit!