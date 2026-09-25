---
layout: post
title: "Project Volterra - ARM Desktop" 
date: 2022-10-30 09:27:00 +0100
comments: false
categories: ARM
---

As a bit of a change from all the PCI/Kubernetes posts, I thought I'd write up my initial impressions of the new [project volterra](https://blogs.windows.com/windowsdeveloper/2022/10/24/available-today-windows-dev-kit-2023-aka-project-volterra/) Windows ARM dev kit, that I got this week. I've been interested in getting an ARM based desktop machine for a while now, but never seen anything that quite hit the mark in terms of performance/pricing.

## Physical and Specification

After unboxing the device, we're left with a fairly small compact system, with a brick style power pack.

![volterra]({{site.url }}/assets/media/volterra.jpg)

So far in use, it's been very quiet and although it's meant to have a fan (as reported [here](https://blogs.windows.com/windowsdeveloper/2022/10/24/available-today-windows-dev-kit-2023-aka-project-volterra/)) I've not heard it in use at all. In terms of power drain, when switched on and left at a Windows desktop, it seemed to be drawing ~4 watts, so not too thirsty.

## Container Tooling Setup

My goal was to set the device up to do containerization work and some development, so I wanted to see what tools would work ok. A first port of call was getting WSL2 up and running so I could SSH directly into that environment. Here I found that the WSL2 setup in the windows app store doesn't currently work with Windows OpenSSH server, so doing a [WSL manual install](https://learn.microsoft.com/en-us/windows/wsl/install-manual) was necessary to get it setup. However once I got that done, it worked pretty well.

Next step was to get Docker up and running. At the moment Docker Desktop doesn't have a Windows ARM build that I can see, so I just installed docker-ce directly inside WSL using [Docker's install process](https://docs.docker.com/engine/install/ubuntu/). The only niggle there is that it needs starting manually with `sudo service docker start`.

After I got Docker running next step was having local Kubernetes clusters with [KinD](https://kind.sigs.k8s.io/), this works out of the box as do any of the Golang tools I've tried so far, as ARM64 on Linux is pretty well supported.

Another tool I use a lot is VS Code and this installed no problems, probably unsurprisingly as Microsoft want this device to be a dev kit :) I had less success with GitHub Desktop though as there doesn't appear to be an ARM64 version available at the moment, so command line `git` is it!

## Other Tools

There's some other things I wanted to potentially use on this device. First one is the note taking tool I use [Obsdian](https://obsidian.md/). Here I was pleasantly surprised  to find that they have a Windows ARM64 build on the download page, which works no problem at all!

Next up was dropbox for file synchronization, and unfortunately as far as I can see there's currently no Windows ARM64 version available.

For remote access, when I'm travelling, I use the amazing [tailscale](https://tailscale.com/) and it installed with no problems at all (from [this](https://github.com/tailscale/tailscale/issues/5218) looks like version 1.32 added the ARM64 package), which is nice. 

## UEFI Settings

I got a request to add the UEFI configuration settings here, so this is the four sets of options when you boot to the BIOS (N.B. you need to use the mini-displayport video out rather than USB-C for this)

![volterra uefi information]({{site.url }}/assets/media/volterra-uefi-information.jpeg)

![volterra uefi security]({{site.url }}/assets/media/volterra-uefi-security.jpeg)

![volterra uefi secure boot]({{site.url }}/assets/media/volterra-uefi-secure-boot.jpeg)

![volterra uefi boot]({{site.url }}/assets/media/volterra-uefi-boot.jpeg)

![volterra uefi management]({{site.url }}/assets/media/volterra-uefi-management.jpeg)

## Conclusion

Project Volterra seems to be a useful dev box for ARM64 work and so far it's working out pretty well. The small form-factor and low power draw make it an attractive option for a headless box that can run a decent range of workloads and, in containerization land anyway, software support is pretty good.

