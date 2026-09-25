---
layout: post
title: "Restricting Docker Access With a Reverse Proxy" 
date: 2021-09-05 09:50:39 +0100
comments: false
categories: Docker
---

One of the things that comes up a lot with Docker security is that, by default, the authorization model is all or nothing. Essentially any user or process that has access to the Docker socket (or Docker TCP port if it's configured to listen on a network) can use any of its functions. This means that from a security standpoint essentially any user with Docker access can get root on the host [easily](https://zwischenzugs.com/2015/06/24/the-most-pointless-docker-command-ever/).

This is a bit of a problem if you have something like monitoring software, which you want to be able to get stats on running containers, but you don't really want it to have access to all of Docker's functionality. So I was wondering, what would be a good way to restrict it, so you can let the monitoring software see the information it needs, without giving it root rights to the host.

## HTTP APIs and Reverse Proxies

As Docker is essentially an HTTP API, one idea would be to use a reverse proxy to sit in front of it and filter what paths are available. Ideally it would be good to have a proxy that understands unix socket files, so we don't need to expose Docker on the network to make it work. Luckily a brief bit of searching showed me there were some good options and in particular [Caddy](https://caddyserver.com/) looked like it would fit the bill. As an aside the initial version of Caddy I tried (v2.4.3) would hang with some Docker responses but the latest version (v2.4.5) should be fine.

Before I get into the details, I'll mention this is just a PoC, I wouldn't rely on it in production without more work and testing, but the basics work fine :)

## Setting up a Docker Reverse Proxy with Caddy

After installing Caddy the first thing to do was make sure it worked the way I expected. For that a simple Caddyfile like this one does the trick

```
{
  debug
}


http://localhost:2379 {
  reverse_proxy unix///var/run/docker.sock
}
```

This will expose a network port of 2379/TCP and just send on any requests to the Docker socket. The `debug` stanza at the top just gets Caddy to output verbose information which is handy when working out what's happening.

With that in place we can just tell Docker to connect to that port and see if it works using the `-H` switch

```
docker -H tcp://localhost:2379 info
```

That works just fine and we can see in the Caddy debug log, the requests going through the proxy to Docker

```
2021/09/05 08:37:07.052 DEBUG   http.handlers.reverse_proxy     upstream roundtrip      {"upstream": "unix///var/run/docker.sock", "request": {"remote_addr": "127.0.0.1:55376", "proto": "HTTP/1.1", "method": "HEAD", "host": "localhost:2379", "uri": "/_ping", "headers": {"User-Agent": ["Docker-Client/20.10.8 (linux)"], "X-Forwarded-Proto": ["http"], "X-Forwarded-For": ["127.0.0.1"]}}, "headers": {"Pragma": ["no-cache"], "Server": ["Docker/20.10.8 (linux)"], "Date": ["Sun, 05 Sep 2021 08:37:07 GMT"], "Api-Version": ["1.41"], "Content-Length": ["0"], "Content-Type": ["text/plain; charset=utf-8"], "Cache-Control": ["no-cache, no-store, must-revalidate"], "Docker-Experimental": ["false"], "Ostype": ["linux"]}, "status": 200}
2021/09/05 08:37:07.067 DEBUG   http.handlers.reverse_proxy     upstream roundtrip      {"upstream": "unix///var/run/docker.sock", "request": {"remote_addr": "127.0.0.1:55376", "proto": "HTTP/1.1", "method": "GET", "host": "localhost:2379", "uri": "/v1.41/info", "headers": {"X-Forwarded-For": ["127.0.0.1"], "User-Agent": ["Docker-Client/20.10.8 (linux)"], "X-Forwarded-Proto": ["http"]}}, "headers": {"Api-Version": ["1.41"], "Content-Type": ["application/json"], "Docker-Experimental": ["false"], "Ostype": ["linux"], "Server": ["Docker/20.10.8 (linux)"], "Date": ["Sun, 05 Sep 2021 08:37:07 GMT"]}, "status": 200}
```

This information is useful because it lets us know what paths we need to whitelist for our restrictive proxy. It's important to try and whitelist the paths you definitely need and not try to blacklist the paths you don't need, otherwise one missing path could ruin your whole day.

With that setup in place, we can just run all the Docker commands we want our monitoring software to do and build out a set of paths that should be allowed by the proxy. My set ended up like this.

```
{
  debug
}

http://localhost:2379 {
  @infourl {
    method GET
    path /v1.41/info
  }

  @pingurl {
    method HEAD
    path /_ping
  }

  @versionurl {
    method GET
    path /v1.41/version
  }

  @eventsurl {
    method GET
    path /v1.41/events
  }

  @statsurl {
    method GET
    path /v1.41/stats
  }

  @containerstatsurl {
    method GET
    path /v1.41/containers/*/stats
  }

  @containersurl {
    method GET
    path /v1.41/containers/json
  }

  reverse_proxy @pingurl unix///var/run/docker.sock
  reverse_proxy @infourl unix///var/run/docker.sock
  reverse_proxy @versionurl unix///var/run/docker.sock
  reverse_proxy @eventsurl unix///var/run/docker.sock
  reverse_proxy @statsurl unix///var/run/docker.sock
  reverse_proxy @containerstatsurl unix///var/run/docker.sock
  reverse_proxy @containersurl unix///var/run/docker.sock
}
```

With that set of paths in place, software should be able to gather stats, list Docker engine information and list the running containers.  If the client tries any other operation it just produces an EOF error.

```
docker -H tcp://localhost:2379 images
EOF
```

## Conclusion

So this is one option for trying to provide a restricted view of the Docker API for things like monitoring tools. Getting to something production ready would obviously require more work, but the basic concept is interesting.