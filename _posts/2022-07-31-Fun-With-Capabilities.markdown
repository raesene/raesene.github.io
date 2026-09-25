---
layout: post
title: "Fun with Capabilities" 
date: 2022-07-31 09:10:39 +0100
comments: false
categories: Kubernetes
---

Capabilities are an interesting area of Linux security and one which has some application to containers. Whilst the details of how they work have been well documented (I'd recommend reading Adrian Mouat's two part series [here](https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work) and [here](https://blog.container-solutions.com/linux-capabilities-in-practice)) I thought it was worth looking at a couple of neat tricks we can use do with file capabilities when using containers.

## Using File Capabilities when you're not root

Sometimes, in containerized environments, we are restricted from running as the root user, a notable example being OpenShift, which restricts that by default. However generally in those cases we'll still get the default set of capabilities that most Linux container runtimes provide. This gives us what are called "bounding" capabilities, but the tricky part is, how do we use those when we're not running as root? 

Well if you can specify a container image that you control, the answer is to set capabilities on programs inside the image, that way when you use those programs you'll be able to use the capabilities granted.

Let's look at a practical example. In the Dockerfile for `raesene/alpine-noroot-containertools` I've got the line

```Dockerfile
RUN setcap 'cap_net_raw,cap_net_bind_service,cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=+ep' /bin/busybox
```

This sets all the capabilities from the Default Docker set on the busybox program inside this container, and busybox provides all the utilities like `chmod` and `chown` so I can run those commands with the capabilities I've specified. There's a couple of useful side effects to doing this. The first is I can change the permissions of files in the image that I don't own

![change file perm inside image]({{site.url }}/assets/media/capability-change-file-perm-in-image.png)

fun, but not really that useful as you can always just change the permissions inside the image while you're building it. A slightly more useful use case is that if you're mounting files inside the container from the underlying host, you don't have to worry about who owns those files or the permissions on them as you can just change the perms, using your file capabilities.

![change file perm inside image]({{site.url }}/assets/media/capability-change-file-perm-on-mounted-dir.png)

This is a handy way to get round the problem where you're running your non-root containers, but don't want to have to remember to change the permissions on files you mount in from the underlying host.

## Moving files with Capabilities

One challenge, when working with file capabilities, is that it can be hard to move them around without losing the capabilities. Utilities like `cp` will generally strip capabilities when they're used. So if you have a file with capabilities how do you move it somewhere?

The answer is `tar` which supports that idea. You can create a tar file using the `--xattrs` flag and then when you un-tar use the `--xattrs-include='security.*'` flag to ensure they're retained. A notable restriction is that to un-tar using that flag, you need to be root. Another restriction is that not all implementations of tar support those flags (e.g. some versions of busybox).

So to demo this we'll use an `ubuntu:22.04` image. Our goal is going to be, starting as an ordinary user on the host with Docker rights, get a copy of `vim` that will let us edit files owned by root like `/etc/passwd` (N.B. this isn't really a security hole as Docker access == root in most cases, but it's fun :) )

![change file perm inside image]({{site.url }}/assets/media/start-container-with-mapped-dir.png)

Once we've got it running first install `vim` and `libcap2-bin` which lets us set file capabilities. With those installed we can run this to set our file capabilities.

```bash
setcap 'cap_net_raw,cap_net_bind_service,cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=+ep' /usr/bin/vim.basic
```

Now we can tar our copy of `vim.basic` up in our mounted directory, and then untar it in the same place (so that the capabilities are preserved)

![change file perm inside image]({{site.url }}/assets/media/tar-file-capabilities.png)

Now, with our copy of vim that has caps ready, exit the container and try to edit a file that we should not have access to on the host, and it should work just fine, thanks to the capabilities applied (specifically `DAC_OVERRIDE`)

To bring it all together here's a video showing all the steps

<iframe width="560" height="315" src="https://www.youtube.com/embed/Ha7PRo1OnCE" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Conclusion

File capabilities are an interesting area to explore when working with containers (and in Linux in general) whilst the security model has been pretty well thought out, there are cases where you can make use of them to allow things that might otherwise be tricky :)


