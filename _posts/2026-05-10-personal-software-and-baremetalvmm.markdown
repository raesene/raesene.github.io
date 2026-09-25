---
layout: post
title: "Personal Software and BaremetalVMM"
date: 2026-05-10 08:00:00 +0100
comments: false
categories: LLMs
---

For a long time I wanted a piece of software that used [Firecracker](https://firecracker-microvm.github.io/) to create MicroVMs on my Linux hosts. It seemed like it would be really useful for vulnerability research and testing features that weren't suitable to be done in Docker containers. I looked around periodically but wasn't able to find anything that really fit the bill and would work easily.

Back in January I was experimenting with [Claude Code](https://code.claude.com/docs/en/overview) and I decided, pretty much on a whim, to see if it could create that software for me. Honestly I didn't think it would work but it would be an interesting experiment to see how far it could get. Surprisingly, after a bit of churning it produced something that, for the basic use case, worked pretty well!

Since then I've kept working on it, having Claude Code expand the features, worked on how to test things (like using [playwright](https://playwright.dev/docs/getting-started-mcp) for browser testing) to the point where now it's got an array of features that are very useful for me. It can start Kubernetes clusters, VMs with different kernels and there's a Web UI and systemd service which mean I can start and stop VMs whenever.

The latest addition was using [xterm.js](https://xtermjs.org/) to give me a browser based console so I can use my VMs remotely without even needing a terminal!

All of this was designed by me, for me, and it fits my use cases pretty well. However it's not been widely tested with other systems and I make it really clear in the README that it's likely only suitable for my use (the code is [on GitHub](https://github.com/raesene/baremetalvmm) in case anyone else wants to try it or use as a basis for something else).

This is a good example of what gets called "personal software"

## The rise of personal software

This idea, that people will write software for their own use using LLMs, is one that's getting quite a bit of traction. Whilst given enough time and effort I possibly could have written my VM manager myself, realistically there's no way I actually had the time to do it. From a personal usage perspective this has been great. I can get tools that do exactly what I'm looking for relatively quickly and easily.

So now it's pretty easy to turn an idea into working software, at least for basic tools like this. The barrier to creation is substantially lowered and so we'll inevitably see more and more similar efforts. Github's [recent blog](https://github.blog/news-insights/company-news/an-update-on-github-availability/) shows the massive increase in activity they're seeing as a result of heavy LLM usage. What's kind of interesting to think about, to me, is what some of the consequences of this trend will be for software security and the general software industry. 

From a security standpoint there are lots of likely challenges here :-

- Whilst LLMs can write software pretty well, they don't necessarily do it with security in mind, and even with code reviewing agents (if people use those) it's likely the security architecture of personal software projects is not going to be great. As an example, while I was writing this blog I realised that the LLM had defaulted to exposing the web UI of BaremetalVMM to all interfaces, which is probably not a good idea (it does have some authentication, but that's not been tested anything like enough to give me confidence to expose it to untrusted networks)!

- Supply chain and maintenance. When you're vibecoding software you probably never look at the libraries that the LLM chose to include, so you have really no idea of what your supply chain risks are, and for a lot of people outside the security industry, I doubt they'd think to look into that problem too much.

- Anyone who's been in IT/IT security for a while will have encountered a "load bearing spreadsheet" or similar. Some system designed by someone who's a subject matter expert but not an IT professional, which has become crucial for a department or whole company's operation. With LLM tools, we're going to see a big increase in this kind of system, and I'd guess a lot of IT teams are going to be handed "personal software" projects to run in production.

In addition to the security concerns, there are also obvious consequences here to how open source projects will work in future. Any time you have groups of people working together, there's inevitable friction with differing priorities and approaches, but traditionally, having sets of people working on a project allowed it to advance much more quickly than a solo project.

That's no longer really the case, now a solo developer with access to LLMs can create an entire project by themselves quite quickly. Their incentives to work with others are changed, and it could be that we'll see a proliferation of projects covering the same topic, each run by a single developer or perhaps a small group. As an example, there are now plenty of projects doing similar things to BaremetalVMM.

## Conclusion

Like lots of things in the AI/LLM world, things are moving pretty quickly in the field of personal software. I definitely think this will carry on as a phenomena as it's solving people's problems, but I'm not entirely sure it'll play out well from a security standpoint. Definitely a case of living in interesting times...