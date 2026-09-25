---
layout: post
title: "Docker Capabilities and no-new-privileges"
date: 2019-06-01 10:10:39 +0100
comments: false
categories: Docker
---


I've been looking for a way to explain an demonstrate the "no-new-privileges" option in Docker for a little while for my [training course](https://www.blackhat.com/us-19/training/schedule/#mastering-container-security-14020) and recently came up with a way that should work, so thought it was worth a blog post.

## Capabilities and Docker

First a little background.  Docker makes use of [capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html) as one of the layers of security that it applies to all new containers. Capabilities are essentially pieces of the privileges that the `root` user gets on a Linux system.  They enable processes to perform some privileged operations without having the full power of that user, and are very useful to get away from the use of `setuid` binaries. Docker applies a restriction that when a new container is started, even if the root user is used, it won't get all the capabilities of root, just a subset.

An important point to note is that, if your process doesn't need any "root-like" privileges, it shouldn't need any capabilities, and processes started by ordinary users don't generally get granted any capabilities.

For more details on Docker and capabilities there's a post [here](https://raesene.github.io/blog/2017/08/27/Linux-capabilities-and-when-to-drop-all/) that goes into some more depth on the topic.

## no-new-privileges

So where does no-new-privileges come into all this?  Well this is an option that can be passed as part of a `docker run` statement as a security option.  The Docker documentation on it says you can use it 
```
If you want to prevent your container processes from gaining additional privileges
```

Basically if your container runs as a non-root user (as all good containers should) this can be used to stop processes inside the container from getting additional privileges.

Here's a practical example.  Say we have a Dockerfile that looks like this

```
FROM ubuntu:18.04

RUN cp /bin/bash /bin/setuidbash && chmod 4755 /bin/setuidbash

RUN useradd -ms /bin/bash newuser

USER newuser

CMD ["/bin/bash"]
```

We've got another bash shell which we've made setuid root, meaning that it can be used to get root level privileges (albeit still constrained by Docker's default capability set).

If we build this Dockerfile as `nonewpriv` then run

```
docker run -it nonewpriv
```

we get landed into a bash shell as the `newuser` user.  Running `/bin/setuidbash -p` at this point will make us root demonstrating that we've effectively escalated our privileges inside the container.

Now if we try launching the same container, but add the no-new-privileges flag

```
docker run -it --security-opt=no-new-privileges:true nonewpriv
```

when we run `/bin/setuidbash -p` our escalation to root doesn't work and our new bash shell stays running as the `newuser` user, foiling our privilege escalation attempt :)

So, this option is one worth considering if you've got containers being launched as a non-root user and you want to reduce the risk of malicious processes in the container trying to get additional rights.

**Edit** Just to add based on conversations on twitter following this post, it's worth noting that you can achieve the same effect as `no-new-privileges`, if you want to run your container as a non-root user, by doing `cap-drop=all` as part of the run statement.  This has a similar effect in stopping the contained processes from gaining any Linux capabilities, and if your workloads support it, is a great idea for hardening your containers.