---
layout: post
title: "Comparing Hyper-V and VMWare Workstation"
date: 2015-07-26 18:28:48 +0100
comments: false
categories: Virtualization
published: false
---

I've been a user of VMWare Workstation for many years (started back on v2 around 2000) and I've always found it extremely useful for experimenting with different OSs and trying things out without mucking up my base build.

However with recent versions of Windows (8/8.1 pro) you get a virtualization hypservisor (Hyper-V) for free, which makes paying £100-200 every couple of years to upgrade to the latest version of VMWare Workstation a bit more of a tricky sell, and at least necessitates looking into how they stack-up.

So this is some notes I've made while I've been trying out Hyper-V with comparisons my experiences with VMWare Workstation 10/11.

## Networking

One thing I've noticed here is that there's a difference in approach between VMWare and Hyper-V.  VMWare creates multiple network interfaces on the host OS for each networking type offered (bridged, NAT, Host only).

By default Hyper-V creates a single Virtual Switch to connect systems to.  You can also create "internal" and "private" Virtual switches for restricted communications.  The item that Hyper-V appears to lack is the NAT network (although there is at least one [workaround](http://thomasvochten.com/archive/2014/01/hyper-v-nat/))

I've used the NAT functionality of VMWare Workstation, so that's a bit of a loss, but not terminal.

## Host/Guest communications

One of the nice parts of Workstation is that there's a number of ways of moving data between the hosts and guests (although this is not without "outages" where bugs prevent some of them working with some guest OSs)

The main ones I use are, Shared folders, which expose a directory on the Host to a given Guest OS and Clipboard support to easily move text from the host to the guest and vice versa.



