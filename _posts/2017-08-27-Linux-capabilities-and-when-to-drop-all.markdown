---
layout: post
title: "Linux Capabilities and when to drop all"
date: 2017-08-27 22:00:39 +0100
comments: false
categories: Capabilities
---

Somewhat following on from my [previous post](https://raesene.github.io/blog/2017/07/23/network-tools-in-nonroot-docker-images/) about running containers in non-root environments I've been spending some more time reading up on Capabilities, so thought it would be worth making some notes.

## What are Capabilities?

Linux capabilities have been around in the kernel for some time.  The idea is to break up the monolithic root privilege that Linux systems have had, so that smaller more specific privileges can be provided where they're required.  This helps reduce the risk that by compromising a single process on a host an attacker is able to fully compromise it.

One point to make note of is, that capabilities are only needed to carry out privileged actions on a host.  If your process only needs to carry out actions that an ordinary user could without the use of sudo, su or setuid root binaries, then your process doesn't need any capabilities assigned to it.

To provide a concrete example, take the `ping` program which ships with most Linux distributions. Traditionally this program has been setuid root due to the fact that it needs to send raw network packets and this privilege is not available to ordinary users.  With a capability aware system this can be broken down and only the CAP_NET_RAW privilege can be assigned to the file.  This means that an attacker who was able to compromise the ping binary, would only get a small additional level of privilege and not full access to the host, as might have been possible when it was setuid root.

## Practical use of capabilities

So how do we actually manipulate capabilities on a Linux system? The most basic way of handing this (without writing custom code) is to use the `getcap` and `setcap` binaries which come with the libcap2-bin package on debian derived systems.

If you use getcap on a file which has capabilities, you'll see something like this

```
/usr/bin/arping = cap_net_raw+ep
```

We can see here that the arping file has cap_net_raw with `+ep` at the end of it, so what does that mean.  The e here refers to the effective capability of the file and the p to the permitted capability. Effectively for file capabilities the effective flag is needed where the binary isn't "capability aware" i.e. it's not written with capabilities in mind (which is usually the case).  For practical purposes if you're assigning capabilities to files, you'll use `+ep` most of the time.

So if you want to assign a capability, for example to apply cap_net_raw to an nmap binary

```
setcap cap_net_raw+ep /usr/bin/nmap
```

It's important to note that you can't set capabilities on symlinks, it has to be the binary, and also you can't set capabilities on shell scripts (well unless you have a super-recent kernel)

## Some More background - Inheritable and Bounded

If you look at capability sets for files and processes, you'll run across two additional terms which bear looking at, Inheritable and Bounded.

Inheritable capabilities are capabilities that can be passed from one program to another.

Bounded capabilities are, to quote the [Man page for capabilities](https://linux.die.net/man/7/capabilities)


> The capability bounding set acts as a limiting superset for the capabilities that a thread can add to its inheritable set using capset(2).


So they restrict which capabilities can be inherited by a process.

## Back to the practical - Auditing capabilities

This is all well and good, but how do we audit capabilities?

there's a number of ways of reviewing what capabilities a process or file has got.  From a low-level perspective, we can review the contents of /proc/[pid]/status.  This will contain some information that looks like this :-

```
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 0000003fffffffff
CapAmb: 0000000000000000
```
  
This set was for a user level process (using the command `cat /proc/self/status`).  As you can see the CapPrm and CapEff are both all zero's indicating that I don't have any capabilities and assigned.  

If I then switch to a root user using `sudo bash` and run the same command, I  get the following

```
CapInh: 0000000000000000
CapPrm: 0000003fffffffff
CapEff: 0000003fffffffff
CapBnd: 0000003fffffffff
```
which is quite the difference, here CapPrm and CapEff have a lot more content as I'm a privileged user.

If we try the same in a Docker process using the command `docker run alpine:latest cat /proc/self/status` we get

```
CapInh: 00000000a80425fb
CapPrm: 00000000a80425fb
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
```

which is quite different.  In this container we were running as root, so you might have guessed that we'd have the same permissions as we did in the root shell before. However as Docker limits the available default permissions we don't get as much.

### Interpreting capabilities

Of course these long hex strings aren't exactly the most friendly way of viewing capabilities.  Luckily there are ways of making this a bit more readable.  if we use `capsh` (which comes with libcap2-bin on debian derived systems) we can work out what's meant here.  

Running `capsh --decode=0000003fffffffff` returns 

```
0x0000003fffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,37
```

So that shows that our root shell run outside basically had all the capabilities.  If we run `capsh --decode=00000000a80425fb` we can see what Docker provides by default

```
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```

which corresponds to the list in the [source code](https://github.com/moby/moby/blob/master/oci/defaults.go#L14-L30).

### Dangerous Capabilities

So which of these capabilities are a concern from a security perspective?  Well to an extent that's going to depend on how you're using them.  However there are some good starting points to look at, firstly there's [this post](https://forums.grsecurity.net/viewtopic.php?f=7&t=2522) from the grsecurity forum that goes into the risks of allowing various capabilities.  You can also look at the default list of capabilities that Docker allows (link above), as the ones they block are things that have been determined as dangerous in the context of containers.

### Other Utilities

There are some other utilities which are handy for doing things like auditing capabilities.  the [libcap-ng-utils](https://people.redhat.com/sgrubb/libcap-ng/index.html) package has the very handy `filecap` and `pscap` programs which can be used to review capapbilties on all files and all processes on a system by default.  There's also `captest` which will review capabilities in the context of the current process.

Also if you're running containers and want a nice quick way to assess capabilities amongst other things, you could use Jessie Frazelle's [amicontained](https://github.com/jessfraz/amicontained)

### Capabilities and Containers

So what has all this to do with Containers?  Well it's worth noting what was mentioned early in this post which is, if you have a container which will run as a non-root user and which has no setuid or setgid root prgrams in it, you should be good to drop all capabilities.  This adds another layer of hardening to the container, which can be helpful in preventing container breakout issues.

If you're running with root containers, then it's well worth reviewing the default list of capabilities that is provided by your container runtime an ensuring that you're happy that these are needed.

Specifically there are ones like CAP_NET_RAW in the default docker set which could be dangerous (see [here](https://www.nccgroup.com/globalassets/our-research/us/whitepapers/2016/june/container_whitepaper.pdf) for more details)


### Capability Gotcha's

There are some gotcha's to be aware of when using capabilities.  First up is that, to use file capabilities, the filesystem you're running from needs have extended attribute (xattr) support.  A notable exception here is some versions of aufs that ship with some versions Debian and Ubuntu.  This can impact Docker installs, as they'll use aufs by default.

Another one is that where you're manipulating files you need to make sure that the tools you're using understand capabilities.  For example when backing up files with tar, you need to use the following switches to make it all work.

```
--xattrs               Enable extended attributes support
--xattrs-exclude=MASK  specify the exclude pattern for xattr keys
--xattrs-include=MASK  specify the include pattern for xattr keys
```

In practice for tar you'll likely want to use `--xattrs` and `--xatttrs-include=security.capability` to make backups of files with capabilities.