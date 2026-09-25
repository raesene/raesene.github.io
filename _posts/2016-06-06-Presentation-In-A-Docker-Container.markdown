---
layout: post
title: "Presenting from a Docker Container"
date: 2016-06-06 10:01:39 +0100
comments: false
categories: Docker Presentations
---

I've been presenting a bit recently on [docker](http://www.docker.com) and in an attempt to keep my presentation environment relatively simple, I decided to move off from using prezi which doesn't have a linux client to something a bit more platform agnostic.

After some looking around I settled on using [jekyll-revealjs](https://github.com/dploeger/jekyll-revealjs) which combines [Jekyll](https://jekyllrb.com/) publishing with the [Reveal.JS](http://lab.hakim.se/reveal-js/#/) HTML/JS presentation framework. 

This turns out to have quite a nice workflow for creating presentations as all your content is a series of markdown files and you can just commit changes to a git repository, which makes updating and modifying content for your presentation very easy, basically anywhere with a text editor does the job.

Also once your presentation is complete you can access from any machine with a browser, which removes the ties of having to run on a specific operating system with specific presentation software installed.

All that said, one slight downside which occurred to me is that whilst editing the presentation is as simple as running a text editor, when you're presenting you need a working jekyll environment with a valid ruby install and the attendant gems. This sounded like a classic case where .... you guessed it.... Docker comes in handy !

So I turned my presentation into a docker container :)

The Dockerfile to do this was really pretty straightforward

```
FROM jekyll/jekyll
MAINTAINER Rory McCune <rorym@mccune.org.uk>
RUN mkdir /presentation
WORKDIR /presentation/
ADD . /presentation/
RUN chown -R 1000:1000 /presentation/*
CMD ["jekyll", "serve", "-H", "0.0.0.0"]
```

from the base jekyll install it basically just adds the git working directory to the image and runs jekyll to serve up the presentation.  The only gotcha I encountered was that the jekyll image very sensibly doesn't run as root, so you need to chown your presentation files appropriately to avoid permission issues when you run the container.

This is also a useful time to use a ```.dockerignore``` file to avoid the container getting all the git history which it doesn't need

```
.git
.gitignore
README.md
```

After that it's just a questions of running the container and exposing the relevant port, and your presentation will run from any machine that has docker installed and a browser to view the content.

In my case it's let me have a backup of my talk sitting on a [web server](http://presentation.pwndland.uk) in case of last minute laptop problems.  I may also have created a Kubernetes cluster to run it, but that's a story for another blog post :)