---
layout: post
title: "WSL and Docker for Windows"
date: 2018-03-29 14:45:39 +0100
comments: false
categories: Docker
---

There's a number of steps needed to get all this setup properly, but at the end of it you should be able to run Linux and Windows containers on a Windows host from WSL bash...

**First up** - Install [WSL](https://docs.microsoft.com/en-us/windows/wsl/install-win10)

**Second up** - Install Docker for Windows from [here](https://store.docker.com/editions/community/docker-ce-desktop-windows).  Stable channel should have all the features you need now, but if you want new stuff quicker, it could be worth using the edge channel.  

**Important 2.5 step** - Once you've installed Docker for windows, you will need to change to "windows containers", by right-clicking the tray icon and choosing "switch to windows containers".  I tried this process without that set, and it didn't seem to work.

**Third up** - Install the docker client binary inside WSL.  Don't use the ubuntu/debian package Docker it's not the right one at all.  Also I'd avoid the docker.io package as it's very outdated.  Instead get a binary from the Docker site [here](https://download.docker.com/linux/static/stable/x86_64/), then extract the `docker` binary and copy to `/usr/local/bin` (or anywhere else on the path that you'd like to keep it)

## The insecure way

At this point you can do this the insecure way and enable Docker to listen on a TCP port without SSL. There's an option in Docker for Windows to allow it to listen purely on localhost, however I wouldn't recommend this as any SSRF style attack on your laptop could result in bad times, as there is no authentication on Docker in this setup, so anyone who can hit the port can do bad things to your system.

## The not horribly insecure way.

Largely based on [this post](http://wslcorsair.blogspot.co.uk/2018/02/secure-nested-lcow-part-2.html) and [this post](http://wslcorsair.blogspot.co.uk/2018/02/secure-nested-lcow-part-3.html) from [Nuno Do Carmo](https://twitter.com/nunixtech).  This is a bit more involved, but more secure.  Basically we're going to configure the Docker daemon to listen on a TCP port but verifying the TLS certificate to provide a level of client authentication.  It's worth noting that Docker's authentication mechanism isn't hugely sophisticated, it's basically just based on "is this certificate signed by a CA I trust", so it's important not to use a CA that's used for a lot of other things, or you could end up with a rather easy to bypass authentication check!

### Setup SSL/TLS on the daemon

The first step is to configure our daemon to listen on TLS.  For that we'll need some certificates.  So first check that you've got openssl installed in WSL, and if not `apt install openssl` should fix it.

Then we're largely going to follow [this process](https://docs.docker.com/engine/security/https/#create-a-ca-server-and-client-keys-with-openssl).

In WSL, go to /mnt/c/ProgramData/Docker/

then `mkdir certs` and `cd certs`

Then we'll generate our CA Key with `openssl genrsa -aes256 -out ca-key.pem 4096`.  Set a decent passphrase that you won't forget :)

Then use `openssl req -new -x509 -days 3650 -key ca-key.pem -sha256 -out ca.pem` to generate the CA certificate.  Feel free to modify the days value up or down depending on how long you want it to be valid.

It'll ask you to fill in some values here, but for the purposes of Docker Authentication these don't matter too much.

Next step is `openssl genrsa -out server-key.pem 4096` to generate our server key.  We then need a CSR which we can generate with `openssl req -subj "/CN=127.0.0.1" -sha256 -new -key server-key.pem -out server.csr`

After that we need to specify some attributes `echo subjectAltName = IP:10.10.10.20,IP:127.0.0.1 >> extfile.cnf` is used to specify valid endpoint IPs.  So change 10.10.10.20 to any other interfaces you want the daemon to listen on.  If you only want localhost (which is all we need here), just remove 10.10.10.20 from the list.

Then set `echo extendedKeyUsage = serverAuth >> extfile.cnf` and generate the signed certificate with `openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf`

After you've done this, you nee to tell the Docker daemon to use your new certificates.  This can be done by editing `C:\ProgramData\Docker\config\daemon.json` .  If this file doesn't exist in that location, you've probably not switched to Windows containers, so do that before editing :)

You'll need to add the following elements to the JSON there `"hosts": ["tcp://127.0.0.1:2376","npipe://"],"tlsverify": true,"tlscacert": "c:\\ProgramData\\docker\\certs\\ca.pem", "tlscert": "c:\\ProgramData\\docker\\certs\\server-cert.pem", "tlskey": "c:\\ProgramData\\docker\\certs\\server-key.pem"`

At that point you should be able to re-start the docker daemon using the tray icon for Windows 10 and it'll be listening on 127.0.0.1:2376

### Setup SSL/TLS in WSL

Now we need to setup the client to authenticate to the server.  

back in `/mnt/c/ProgramData/Docker/certs/` create a key with `openssl genrsa -out key.pem 4096` , then generate a CSR with `openssl req -subj '/CN=client' -new -key key.pem -out client.csr`, add an extended attributes file with `echo extendedKeyUsage = clientAuth >> client-extfile.cnf` and generate the certificate with `openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile client-extfile.cnf`.

Then create a .docker directory in your home directory and  copy key.pem, ca.pem and cert.pem into that directory.

Then run `export DOCKER_HOST=tcp://127.0.0.1:2376` and `export DOCKER_TLS_VERIFY=1` and you should be able to do `docker info` and have it work!

For persistent add those two environment variables into your WSL profile by modifying .bashrc or similar with those commands.

After this the only caveat is, that as you're in Windows containers mode, you need to add `--platform linux` to any Docker commands for linux containers that you run.

Thanks to [Nuno Do Carmo](https://twitter.com/nunixtech) this is just a modified version of a setup he described in the blog posts linked above.