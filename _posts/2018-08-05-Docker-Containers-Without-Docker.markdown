---
layout: post
title: "Docker containers without Docker"
date: 2018-08-05 12:10:39 +0100
comments: false
categories: Containers
---

Following on from looking at [katacontainers](https://raesene.github.io/blog/2018/07/23/exploring-kata/) and [gVisor](https://raesene.github.io/blog/2018/07/22/exploring-gvisor/), I thought it might be interesting to look at the [containerd](https://containerd.io/) project and the idea of using containerd and runc without docker to run containers.  Looking round the documentation, I couldn't find a good look at getting containerd and runc setup together without installing Docker, so lets do that.

## Installation Notes

For this install we're working from a default Ubuntu 18.04 server install.


### Step one - Get Containerd
First step is to get the containerd binaries, they're available on the release page of the containerd github site [here](https://github.com/containerd/containerd/releases)

`wget https://github.com/containerd/containerd/releases/download/v1.1.2/containerd-1.1.2.linux-amd64.tar.gz`

`tar -xzvf containerd-1.1.2.linux-amd64.tar.gz`

`sudo cp bin/* /usr/local/bin/`

This should place the binaries in a location on the path.  In addition to the daemon files there's a copy of the `ctr` binary which is used as a client.

### Step two - Get runc

We can get the runc binary from their Github page [here](https://github.com/opencontainers/runc/releases).  We'll get the AMD64 binary and put it in the same dir as the containerd files

`wget https://github.com/opencontainers/runc/releases/download/v1.0.0-rc5/runc.amd64`

`sudo cp runc.amd64 /usr/local/bin/runc`

`sudo chmod +x /usr/local/bin/runc`

### Step three - Containerd configuration

Now we'll need to provide containerd some configuration and setup the systemd entry so that we can start it automatically on boot.  Containerd has a handy command for generating a default configuration , so we can use that.

`containerd config default > config.toml`

`sudo mkdir /etc/containerd`

`sudo cp config.toml /etc/containerd/`

Next up, setting up systemd.  The containerd project provides a systemd unit file on their github repo. so we can get that and use it.

`wget https://raw.githubusercontent.com/containerd/containerd/master/containerd.service`

`sudo cp containerd.service /etc/systemd/system`

`sudo systemctl daemon-reload`

`sudo systemctl start containerd`

If you want containerd to start on boot you can also add

`sudo systemctl enable containerd`

## Running containers with Containerd

If all has gone well we should now have a running containerd/runc setup, so next up is getting and running a container.  This is a little more involved than the easy Docker process, and there's a couple of different command to know, but nothing too heavy.

First up we can check that the client can connect to the daemon ok with `sudo ctr version` .  If this works, you should see something like

```
Client:
  Version:  v1.1.2
  Revision: 468a545b9edcd5932818eb9de8e72413e616e86e

Server:
  Version:  v1.1.2
  Revision: 468a545b9edcd5932818eb9de8e72413e616e86e
```

Pulling an image is done with `ctr image pull`.  An important note is that unlike Docker this doesn't hard code a default registry, so you need to specify the Docker hub URL if you're pulling from there. `sudo ctr image pull registry.hub.docker.com/library/alpine:3.7` should download an Alpine 3.7 image.

Next up we need to create a container.  This is done with `ctr container create` and you just pass the image and a name so something like `sudo ctr container create -t registry.hub.docker.com/library/alpine:3.7 myfirstcontainer` should work. One point to note is that we're passing the `-t` switch here to provide a TTY to the container

Now at this point you might be wondering "hey why am I not in my container?".  In containerd land you need to start the task after creating the container, so `sudo ctr task start myfirstcontainer` should work and put you in your container.

Looking around you'll notice that whilst most things are similar to where you were running with docker, there are some differences, notably that you've got no networking past the `lo` interface. That's provided by Docker.  You can work round this using `--net-host` on the container create statement to get access to the host's network, but that's a little hacky. Past that it should be possible to hook up other container networking solutions to this, but that's a topic for another blog post!
