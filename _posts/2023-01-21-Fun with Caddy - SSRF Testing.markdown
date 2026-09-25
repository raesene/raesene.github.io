---
layout: post
title: "Fun with Caddy - SSRF Testing" 
date: 2023-01-21 09:27:00 +0100
comments: false
categories: Kubernetes
---

Recently I've been looking at [SSRF in Kubernetes](https://raesene.github.io/blog/2023/01/02/Fun-with-SSRF-Turning-the-Kubernetes-API-server-into-a-port-scanner/). When testing for SSRF, I find it very useful to have a webserver/reverse proxy that I control and can configure to do a number of tasks. I've been using [Caddy](https://caddyserver.com/) for this. In this post I'll show you how to use Caddy to test for SSRF.

## Setting up Caddy

The first benefit I found, from a researcher/pentester standpoint was ease of installation. Caddy is written in Golang and, in common with most Golang utilities, runs as a single binary. Whilst there are more complex installation methods available you can just grab a binary from the [releases page](https://github.com/caddyserver/caddy/releases/tag/v2.6.2) extract the archive (checking signatures/checksums ofc!) and then run it.

## Configuring Caddy

There's two ways to configure Caddy, [JSON and Caddyfile](https://caddyserver.com/docs/getting-started#json-vs-caddyfile). For the kind of simple examples I'm using for SSRF, Caddyfiles are generally a better option as the file is more readable.

In the general case if you have a file called `Caddyfile` in a directory, you can start the proxy with `caddy run`. If you want to use a different file, you can use the `--config` flag. For example, if you have a file called `myconfig` in the current directory, you can start the proxy with `caddy run --config myconfig`.

## Caddy logging

The first setup I wanted was just to have a log of any requests hitting a port. Handy for determining whether a request is causing an SSRF. Having a server listening on all interfaces on port 80, responding with a fixed string and logging the output looks like this. Note that the `log` directive is inside the `:80` block, as we're logging request to that port.

```
# Simple logger for access to port 80
:80 {
  respond "Hello World"
  log {
    output file 80access.log
  }
}
```

Once you've got that, just start it with `sudo caddy run` and you should see a file called `80access.log` in the current directory. If you hit the server with a request, you should see something like this in the log file:

```
2022/12/27 16:34:42.141 info    http.log.access.log0    handled request {"request": {"remote_ip": "217.155.25.114", "remote_port": "19000", "proto": "HTTP/1.1", "method": "GET", "host": "my.test.server", "uri": "/", "headers": {"Accept-Language": ["en-GB,en;q=0.5"], "Accept-Encoding": ["gzip, deflate"], "Connection": ["keep-alive"], "Upgrade-Insecure-Requests": ["1"], "User-Agent": ["Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:108.0) Gecko/20100101 Firefox/108.0"], "Accept": ["text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8"]}}, "user_id": "", "duration": 0.000073504, "size": 11, "status": 200, "resp_headers": {"Server": ["Caddy"], "Content-Type": ["text/plain; charset=utf-8"]}}
```

## Running on multiple ports and TLS

Adding new ports is just a matter of having another port block in the Caddyfile. For example adding a port 443 block looks like this:

```
# Simple logger for 443 with valid TLS
my.test.server:443 {
  tls my@email.address
  respond "Hello Secure World"
  log {
    output file 443access.log
  }
}
```

With this directive I've specified a host `my.test.server` and the `tls` directive which takes your e-mail address as an argument. The combination of these two settings unlocks a fun trick, which is that caddy will automatically request a TLS certificate for the host you specify (assuming of course that you point the DNS A record at the machine running the server and it's Internet facing).

This is very useful for services that require a valid TLS certificate.

## Internal TLS and "on-demand"

Of course sometimes you'll be testing on an internal network, so that setup won't work. If you don't need valid TLS certs you can use the `internal` directive to generate self-signed certs. You can then add the `on_demand` directive to have Caddy generate a cert for any SNI host that's requested.

```
:443 {
  tls internal {
    on_demand
  }
  log {
    output file 443access.log
  }
}
```

It's worth noting you can use `on_demand` on an internet facing host too and Caddy will request a valid TLS cert for any host name that points at that server, but it's not advised as it can cause rate limiting issues with Let's Encrypt.

## Reverse Proxy

So Caddy makes a nice way of handling things like TLS and providing simple responses. Sometimes you might want to use it as a front-end for a webserver running in some other language (e.g. Ruby). Setting Caddy as a reverse proxy is pretty easy just add a line like `reverse_proxy :8080` and replace `:8080` with the port your webserver is listening on.

## Redirect

A handy technique for SSRF can be where you want to convert an initial request which is a POST/PUT to be a GET request. You can do this with Caddy using the `redir` option. For example if you want to redirect a request to `http://169.254.169.254`

```
# Redirect to metadata server
my.test.server:8444 {
  tls my@email.address
  redir http://169.254.169.254
  log {
    output file 8444access.log
  }
}

```

## File server

You can also use Caddy to serve files from a specific directory with the `root` and `file_server` directives.

```
:80 {
  root * /my_files
  file_server
  log {
    output file access.log
  }
}
```

## Respond with JSON

you can use the `respond` directive and one thing I wanted to do was respond with a JSON object. This is pretty easy, just use the `respond` directive and then specify the content type and the JSON object, and the content type is `application/json`. One trick to note is that I needed to remove all spaces in the response for it to work.


## Debug

As with any testing things don't always go to plan, so adding the debug directive at the top of a Caddyfile (not in a port block) will give you a lot more information about what's going on.

```
{
  debug
}
```

## Conclusion

This is just a small section of the things you can do with Caddy, and I'd recommend reading the [docs](https://caddyserver.com/docs/) for more ideas, but hopefully it's useful to people wanting to have simple servers for SSRF testing :)
