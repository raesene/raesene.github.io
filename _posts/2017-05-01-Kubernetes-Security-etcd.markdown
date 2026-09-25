---
layout: post
title: "Kubernetes Attack Surface - etcd"
date: 2017-05-01 15:15:39 +0100
comments: false
categories: Kubernetes
---

[etcd](https://coreos.com/etcd) is a key element of most Kubernetes deployments as it stores the cluster state including items like service tokens, secrets and service configurations.

So keeping access to this limited is pretty important for a secure cluster.  Depending on how your distribution of Kubernetes sets things up, there's a number of different default configurations you might see.

Some, like kubeadm, will bind etcd to the localhost interface only.  In this kind of setup an attacker would need to get access to the master node in order to get access to the API interface, so the exposure is somewhat limited.  

However the problem with localhost binding only is that it doesn't really allow for clustered etcd setups.  If you want to have multiple etcd databases to allow some redundancy you need to allow for communications between datastores.

In these cases port 2379/TCP and 2380/TCP are likely to be exposed on the network.  2379 is for client --> etcd communications, and 2380 is for communications between the different nodes in the etcd cluster.

Its at this point that you'll want to be well acquainted with the CoreOS guidelines on [etcd security](https://coreos.com/etcd/docs/latest/op-guide/security.html).  This lays out the options that are available.  Basically etcd uses client certificate authentication, but there's a couple of important points to note

- There's no checking of information in the certificate CN or SAN fields, so any valid certificate will allow access. So its probably worth using a dedicated certificate authority for the etcd cluster, and not using certificate issued by another CA (such as the one you're using for general Kubernetes setup).

- With etcd and Kubernetes the setup is all or nothing, there's no authorisation used, so be very careful with which clients are allowed access to the etcd datastore.

So say you're reviewing a cluster and want to assess the etcd security posture, what's the approach?

You'll likely need a copy of etcdctl to query the service.  Older versions can be queried with curl, but in newer Kubernetes installs, they've moved to gRPC and curl doesn't work any more :(

etcdctl can be acquired by downloading an etcd release like [this one](https://github.com/coreos/etcd/releases/download/v3.1.5/etcd-v3.1.5-linux-amd64.tar.gz) and getting it from the tarball.  Alternatively if you can deploy containers to the cluster, you could deploy something like [this image](https://hub.docker.com/r/raesene/alpine-containertools/) which has it already installed.

once you've got etcdctl installed, you can query the API with something like

`etcdctl --endpoint=http://[etcd_server_ip]:2379 ls`

If you get back `/registry` you're likely dealing with a v2 install (Kubernetes 1.5 or lower) and you can easily wander through and explore the config.  In particular the `/registry/secrets/default` path is likely to be of interest as it may contain the default service token which can provide elevated rights to the cluster.

If you get back a blank line from the initial query its reasonably likely that you've got a v3 cluster and getting the data out is a bit different.

First up you need to let etcdctl know that you're dealing with v3, so `export ETCDCTL_API=3` is needed.

Once you've got that environment variable set, you should see a different set of etcdctl commands as being available, including `etcdctl snapshot save` .  You can use this command to dump an instance of the etcd database to a file on disk.

This database is in the boltdb format, so it's possible to read the file using something like [boltbrowser](https://github.com/br0xen/boltbrowser).  Unfortunately the format of the data will be a bit broken as it's serialized in proto format (more details in [this github issue](https://github.com/coreos/etcd/issues/7723)), but you can likely still extract some useful information, if that's your goal.

