---
layout: post
title: "Using 'Try with PWD' buttons to demonstrate apps"
date: 2018-10-21 15:10:39 +0100
comments: false
categories: Docker
---

I came across a very interesting post [this morning](https://medium.com/@patternrecognizer/how-to-add-a-try-in-play-with-docker-button-to-your-github-project-41cb65721e94) on [using Play With Docker](https://labs.play-with-docker.com/) (PWD) to let people try out applications directly from your GitHub repository.  If you've not tried out Play With Docker before (or it's companion site, [Play with Kubernetes](https://labs.play-with-k8s.com/)), they're very useful resources which let you try things out in disposable Docker and Kubernetes environments.  Handy for training courses amongst other things.

What I hadn't realised before was that you can pass a Docker compose file in as a parameter to a PWD URL and have it automatically spin up an instance of that stack.  This model seems super-useful for trying out new applications in disposable environments and works well with web applications, as we'll see.

So having read the post I thought I'd try adding a sample instance for my [Dockerized Security Tools](https://github.com/raesene/dockerized-security-tools) project.  From the tools I've got in there at the moment, the best candidate for a try out looked to be [Dradis-CE](https://dradisframework.com/ce/).  

I knocked up a very basic Docker compose file for it, and then put a button referencing it in the Readme, with this as the result.

[![Try in PWD](https://raw.githubusercontent.com/play-with-docker/stacks/master/assets/images/button.png)](https://labs.play-with-docker.com/?stack=https://raw.githubusercontent.com/raesene/dockerized-security-tools/master/dradis/docker-compose.yml)

If you click that button and then login (you'll need a Docker Hub account for this, but they're free to create), PWD will launch the application.  The only other trick to trying it out is that you need to click a link in the PWD interface to access the exposed application.

There will be a grey oval next to the IP address at the top with the exposed port number (in this case 3000).  Clicking that link should take you into a running instance of Dradis! The first page load will be a little slow, but after that it should work just fine.

I could see this having a number of use cases, things like running up instances of [OWASP Juice Shop](https://www.owasp.org/index.php/OWASP_Juice_Shop_Project) to try out tools or similar.

The runtime is limited to four hours, but that should be plenty for a quick look round a tool to see what it's like.