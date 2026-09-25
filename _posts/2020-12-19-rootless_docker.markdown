---
layout: post
title: "Exploring Rootless Docker" 
date: 2020-12-19 17:00:39 +0100
comments: false
categories: Docker
---

With the release of Docker 20.10, the rootless containers feature has left experimental status. This is an important step for Docker security as it allows for the entire Docker installation to run with standard user prvivileges, no use of root required. Other container solutions like Podman have had this feature for a while but if your used to Docker's approach it's nice to see it being available.

Docker's documentation on [rootless containers](https://docs.docker.com/engine/security/rootless/) has some information about how this is achieved, but I thought it'd be interesting to have a poke around some of the details of the implementation and what it means for container security, especially as I'll be adding this to the container security course I do.

## Install and Use

Setting up rootless containers is pretty straightforward, on Ubuntu at least. You need a couple of packages to be installed (the main one you'll likely need to add is uidmap) and then you can use Docker's install script to set it up. Obviously I'd recommend downloading and reading the script rather than following their suggestion to pipe it straight to `sh` but looking through it you'll see it's mainly just checking the environment before setting up and then downloading and extracting the necessary binaries to `$HOME/bin`.

Once it's installed and you tell the docker CLI where to find the socket file (in `/run/user/UID/docker.sock` instead of the usual `/var/run/docker.sock` ) you can start using Docker. The general use case is pretty much exactly what you'd expect, docker commands work fine, you can pull and run images, execute shells in them, and do most of the other things you'd usually do.

A couple of limitations are present, specifically no cgroups support, unless you enabled cgroupsv2 at the host level and no use of ping or ports < 1024, but again these can be configured if needed. I'd expect the primary use case for rootless docker to be on shared development boxes and perhaps CI hosts, so these limitations probably are deal breakers and in both cases simple configuration options are available.

## Exploring under the covers

So what's going on here, and how does it compare to standard Docker. 

### User namespaces

One of the common security challenges when using containes is that they often run as the root user (uid 0) on the host. Whilst Docker has various layers of security to reduce the risk of this, it's still a cause of potential security problems.

So now we're running Docker without any root acccess, what happens?

With rootless docker setup, if we do something like `docker run --name=rootlessweb -d nginx` to start up a container, then run `docker exec rootlessweb whoami` , you'll get back the answer `root`, which shows that the container thinks it's running as root.

To see what's really happening, we can get the PID of the container with `docker inspect -f '{% raw %}{{.State.Pid}}{% endraw %}' rootlessweb` and then look for that process ID in the output of `ps -ef`. What you'll see is that instead of running as root, it's running as your user!

The way this is being done is through the use of [user namespaces](https://man7.org/linux/man-pages/man7/user_namespaces.7.html) which have been available in Docker (but rarely used) for quite some time. Docker is mapping uids inside the container, to different uids outside the container (which is why it needed the uidmap package installed). This allows the contained process to act as though it had root rights, without it actually having root rights to the underlying machine. 

Obviously from a security standpoint this is a big positive as it means that kernel bugs that need real `uid 0` to work, will be blocked as, from the perspective of the kernel, the process is unprivileged.

## Capabilities

One of the things Docker does as part of its setup is use various layers of isolation on contained processes. One of those layers is capabilities. With this one, you might think it's not going to apply to rootless containers as capabilities are often described as "piece of root rights" which don't apply here.

However, reading the user namespace manpages, we can see that actually capabilities are still used inside user namespaces to restrict access. The important part to remember from a security standpoint is that capabilities in user namespaces can only grant rights to resources governed by that namespace. So having CAP_SYS_ADMIN in a user namespace will get you rights in that namespace, but not rights over the underlying host kernel.

We can see the capability setup by running `pscap` on a host running rootless containers. For example the nginx container from the previous example will look like this

```bash
8198  8219  rorym       nginx             chown, dac_override, fowner, fsetid, kill, setgid, setuid, setpcap, net_bind_service, net_raw, sys_chroot, mknod, audit_write, setfcap
```

## AppArmor

This one is not supported under rootless containers, so no profiles will be loaded (unlike standard rootful containers).

## Seccomp filter

The standard Docker seccomp filter is enabled when using rootless containers, which you can see by inspecting the proc status for our contained process with `cat /proc/[PID]/status | grep Seccomp` which should return a value of `2` showing there's a seccomp profile applied.

## Container breakout

So now we've looked at the various layers of isolation that are used in docker and how they apply (or don't) to rootless containers, lets look at some of the practical tools and techniques and what they return when run in a rootless container.

### The most pointless docker command ever

[My favourite docker command](https://zwischenzugs.com/2015/06/24/the-most-pointless-docker-command-ever/) is a good place to start. Running 

```bash
docker run -ti --privileged --net=host --pid=host --ipc=host --volume /:/host busybox chroot /host
```

Generates an error. Both `--pid=host` and `--ipc=host` don't work with rootless containers, after removing those we get dropped to a root shell with access to what *looks like* the root filesystem of the underlying host. Trying to modify things, however, shows we're not in the real root filesystem. We can add files in system directories like `/etc` , however exiting the container shows that the files haven't actually been created, so it's obvious that our volume mount didn't have the effect it would have had, if we had been running a rootful container.

### amicontained

Running [amicontained](https://github.com/genuinetools/amicontained) from inside a rootless container shows that it's easy to detect that we're in a user namespace, and indeed information is available about how UIDs are being mapped inside the container

```bash
Container Runtime: docker
Has Namespaces:
        pid: true
        user: true
User Namespace Mappings:
        Container -> 0  Host -> 1000    Range -> 1
        Container -> 1  Host -> 100000  Range -> 65536
AppArmor Profile: unconfined
Capabilities:
        BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: filtering
Blocked Syscalls (64):
        MSGRCV SYSLOG SETSID USELIB USTAT SYSFS VHANGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPERM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFSSERVCTL GETPMSG PUTPMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES FUTIMESAT UNSHARE MOVE_PAGES UTIMENSAT PERF_EVENT_OPEN FANOTIFY_INIT NAME_TO_HANDLE_AT OPEN_BY_HANDLE_AT SETNS PROCESS_VM_READV PROCESS_VM_WRITEV KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD PKEY_MPROTECT PKEY_ALLOC PKEY_FREE IO_PGETEVENTS RSEQ
```

### botb

Using [botb](https://github.com/brompwnie/botb) to try and autopwn out of a container that has the docker socket mounted runs us into a problem, which is that it tries to use `--pid=host` and `--ipc=host` as part of the breakout. 

This is, in general, a possibly interesting point about container breakout tools, as they're unlikely to take account of rootless containers.

## Conclusion

Rootless containers are a great addition to Docker's repetoire. In situations where you want to have users run docker commands without giving the root access to the underlying host, this will really help out. The setup and usuability are great, but you need to take account of the fact that certain things just won't work the same way as functions reserved for the root user on the host machine won't be usuable.