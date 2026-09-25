---
layout: post
title: "Docker Hub - Watch out for old images"
date: 2018-08-12 12:10:39 +0100
comments: false
categories: Containers
---

One of the key elements of the success of Docker is the availability of Docker Hub, which provides an effective "app store" of pre-build Docker images with a huge variety of pre-installed software.  Everything from Databases, to CRM software to hacking tools is easily available at the drop of a `docker run` command.

However, like any user maintained repositorry, users need to be careful that what they're using meets their needs.  Outside of the "official" Docker images, Docker themselves don't take any responsibility for maintaining images pushed to Docker hub, so users are own their own to determine whether an image is secure and up to date.

It's that second point that I wanted to touch on here, as I'd noticed it recently while working on a project to create [Docker images for common security tools](https://github.com/raesene/dockerized-security-tools) .

If you do a `docker search` for common security tools you get quite a few hit.  for example if you do `docker search metasploit` you the a set of results with these at the top

```
NAME                                            DESCRIPTION                                     STARS  AUTOMATED
linuxkonsult/kali-metasploit                    Kali base image with metasploit                 63     [OK]
remnux/metasploit                               This Docker image encapsulates Metasploit Fr…   44     [OK]
strm/metasploit                                 Metasploit image with steroids (nmap, tor an…   16     [OK]
metasploitframework/metasploit-framework        metasploit-framework                            8      [OK]
vulnerables/metasploit-vulnerability-emulator   Metasploit Vulnerable Services Emulator !       4      [OK]
```
The search function provides a metric in a number of "stars" which can provide an indication of which image is considered the most popular.

Unfortunately what `docker search` doesn't tell you is, when was this image last updated. In this case we can see the following results by checking on Docker hub

```
NAME                                            LAST UPDATED
linuxkonsult/kali-metasploit                    2 years ago
remnux/metasploit                               2 years ago
strm/metasploit                                 9 months ago
metasploitframework/metasploit-framework        3 days ago
vulnerables/metasploit-vulnerability-emulator   9 months ago
```

So the obvious choice (well once you've checked Docker Hub) is the recent image, which also looks like it's maintained by Rapid7 themselves, but if you'd chosen the "most popular" image you'd be well out of date.

In my case what I'm doing is creating images that I control and then using [Microsoft Flow](https://flow.microsoft.com/en-us/) to automate the process of weekly rebuilds (more information on the automated rebuild process [here](https://raesene.github.io/blog/2017/07/09/Keeping-your-Docker-Builds-Fresh/)).  Personally I think controlling your own images is worth the effort as then you've got more confidence on what's included, and once you've got the automated rebuilds working, you've got a better level of confidence that you won't be getting really outdated versions of the software.