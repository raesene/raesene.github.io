---
layout: post
title: "Software Library Repositories and Security"
date: 2015-03-01 21:05:43 +0000
comments: true
categories: Development
---

Last week I did a presentation for the [Securi-Tay Conference](http://www.securi-tay.co.uk).  The title of the talk was "Security and 'modern' software development", and the main theme of the talk was looking at library repositories like [Rubygems](http://www.rubygems.org), [npm](http://www.npmjs.org) and [NuGet](http://www.nuget.org) and how an attacker could try and place malicous content into those locations.

Now nothing in this talk is particularly new as people have been talking about malicious library installs in [Ruby](http://lanyrd.com/2013/rulu/scgxzr/) and [npm](http://webinos.org/2013/06/17/reflections-on-nodejs-malware/) amongst others for some time, however I do think that it's still not a widely recognized problem and particularly as these repositories grow in size and use, we're going to see more malicious content on them.

An example of a fairly basic typo-squatting approach on PyPi was actually seen recently [as noted on /r/python](https://www.reddit.com/r/Python/comments/2wr93b/this_one_looks_odd_doesnt_it/), which lasted for a while before it was noted.

My initial reviews looked at areas like how package maintainers are authenticated to the repos, whether digital signing is supported and/or widely used, whether there is any curation on the repositories and some other associated areas.

In most cases the picture at the moment isn't particularly bright from a security standpoint, although initiatives like [The Update Framework](http://theupdateframework.com/) are trying to improve the situation, progress appears pretty difficult to make.  This isn't particularly surprising as adding security is going to make development/publishing more awkward and at the moment I don't think many developers see this trade-off as worth the added security benefits.

Anyway I have an expanded version of this talk planned for [OWASP AppSec EU](http://2015.appsec.eu/) in May, so I've got some experiments to try out between now and then to look at other aspects of Repository security.

I mention this specifically as I'm going to put a reference to this post into the description of various test libraries that I'm planning to write and if you've installed one and are wondering what the heck I'm up to, this will hopefully provide some information :)