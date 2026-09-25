---
layout: post
title: "Traefiking in Presentations"
date: 2019-03-25 21:10:39 +0100
comments: false
categories: Docker
---

One of the common tasks in a containerized environment, where you could be running multiple applications in containers on a single host, is "what's the best way to route traffic to my web applications"?

Whilst you could expose each application on a dedicated port, that's not a great user experience.  A better idea is to make use of a reverse proxy (a.k.a ingress in Kubernetes-land) to route traffic to the correct container based on a set of rules.

Recently I noticed that [traefik](https://traefik.io/) had a new version on the horizon, so I decided to give it a spin.

My use case is pretty straightforward.  I've got a number of presentations that I've given over the last couple of years which are all bundled as Docker images (using [jekyll-revealjs](https://github.com/dploeger/jekyll-revealjs)). I like this method of writing presentations, as it lets me create the content in a series of markdown files, which are nice and easy to edit, and then bundle up the resulting presentation as a web application runnning in a Docker image, meaning it can be delivered from any Internet connected host.

Traefik configuration can be done using a Docker compose file, where labels in the service definition let traefik know how to route specific requests.  My planned layout was just to specify hosts on my domain (pwndland.uk) for each presentation.

The main "Gotcha" I encountered when writing the configuration below was that I needed to explicitly specify the port on the container network that was hosting the application using `expose` statements, otherwise traefik wouldn't know how to route traffic.  

One note from a security perspective if you're making use of Traefik, is that it

The configuration for my presentations ended up looking as below, and the presentations should be available on these URLS

- [Le Tour Du Hack Presentation - A security Persons life or, "The art of being Cassandra"](http://ltdh.pwndland.uk/#/)
- [44Con March 2019 - Cloudy Clusters Catastrope?](http://kube.pwndland.uk/#/)
- [Cloud Native Glasgow - Fistful of Container Runtimes](http://container-runtime.pwndland.uk/#/)
- [BSides London 2016 - Docker - Security Myths, Security Legends](http://docker.pwndland.uk/#/)



```
version: '3'

services:
  reverse-proxy:
    image: traefik:v2.0 # The official v2.0 Traefik docker image
    command: --providers.docker # Enables the web UI and tells Traefik to listen to docker
    ports:
      - "80:80"     # The HTTP port
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # So that Traefik can listen to the Docker events
  ltdh:
    image: raesene/ltdh_presentation
    expose:
      - "4000"
    labels:
      - "traefik.http.routers.ltdh.rule=Host(`ltdh.pwndland.uk`)"
      - "traefik.domain=`pwndland.uk`"
  kube:
    image: raesene/kube2019pres
    expose:
      - "4000"
    labels:
      - "traefik.http.routers.kube.rule=Host(`kube.pwndland.uk`)"
      - "traefik.domain=`pwndland.uk`"
  container-runtime:
    image: raesene/container-runtime-presentation
    expose:
      - "4000"
    labels:
      - "traefik.http.routers.container-runtime.rule=Host(`container-runtime.pwndland.uk`)"
      - "traefik.domain=`pwndland.uk`"
  docker:
    image: raesene/bsides_presentation
    expose:
      - "4000"
    labels:
      - "traefik.http.routers.docker.rule=Host(`docker.pwndland.uk`)"
      - "traefik.domain=`pwndland.uk`"
```