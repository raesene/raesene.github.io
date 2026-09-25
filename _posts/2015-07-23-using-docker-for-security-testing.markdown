---
layout: post
title: "Using Docker for Security Testing"
date: 2015-07-23 20:56:47 +0100
comments: false
categories: Docker
---

Following on from my [previous post](http://raesene.github.io/blog/2015/07/05/some-notes-on-docker/) about Docker, I've been giving some thoughts to how I could make use of this in my day-to-day work of security testing.

There's a couple of areas where I can see Docker being quite useful, mainly due to the ease of maintaining and installing applications and also the reduced resource utilization over "tradtional" virtual machines.

The first one is in isolating security testing applications which require a bit of setup or a particular environment to operate in.  Some very useful security testing tools are slightly more complex to setup and run than just doing a  `git clone` of the repo and running a script and there can always be conflicts in the version of underlying tools and libraries required for them to run. These tools can benefit from the isolation that using a containerized version brings.

Already on [Docker Hub](https://hub.docker.com) you can see a number of containers avaiable which do this for you.  An example is [OpenVAS](http://www.openvas.org/) for which there are a number of containers available, for example [this one](https://registry.hub.docker.com/u/mikesplain/openvas/).

One of the nice things about containers built using the automated build approach is that Docker Hub will show you the dockerfile used to build the container, which provides some level of transparancy over what you're downloading (and the syntax is relatively basic, so easy to see what's going on).  You still obviously have to trust Docker Hub and the sources of data used to build the container, which may not be applicable to some environments, and there a better option might be an internal docker repository with internally validated builds.  Even there, the Docker Hub examples are useful to base your Dockerfiles on.

Another way in which containers can be handy is in creating new, consistent, security test environments for each piece of work done.  A common problem in security testing is ensuring that your testing environment is clear of any data from previous reviews, to avoid inadvertant contamination and also ensuring that your tools are regularly updated.

Using a containerized image as your base here is quite straightforward as you just get an image, and then create a new container based on that image for each review.  I've started a sample one [here](https://registry.hub.docker.com/u/raesene/sectest/) which shows some of the basic tools.  The Dockerfile is on github [here](https://github.com/raesene/sectest), and could be useful as a starting point for others.

Now you could replicate this with virtual machines, but the downside there is that the disk space requirements for a new VM per test would be pretty hefty and cloning/starting a new VM can be a bit slow, when compared to containers.  Depending on the type of testing you do, you may need the more complete environemnt that a VM provides, but I reckon for a lot of  testing a container should work just fine.

Another potential advantage of this approach is that you could archive off the container at the end of the test which could make reproducing findings at a later date simpler (less risk of tools updates changing the results of a test)

## Some Other Security Testing Containers

As I mentioned before there are quite a few containers turning up on Docker Hub which could be useful, here's some of the ones I've seen so far

* [Dradis community](https://registry.hub.docker.com/u/raesene/auto-docker-dradis/)
* [OWASP ZAP](https://registry.hub.docker.com/u/owasp/zap2docker-stable/)
* [Kali Linux](https://registry.hub.docker.com/u/kalilinux/kali-linux-docker/)
* [WPScan](https://registry.hub.docker.com/u/wpscanteam/wpscan/)
