---
layout: post
title: "Do containers still contain?"
date: 2026-06-03 08:00:00 +0100
comments: false
categories: Containers
---

The questions of whether containers really contain has been an active topic of debate since pretty much as long as containers have been in use and the answer, like most things in security, is it depends! Security isn't an absolute but calculations do change with new threats and tools and I think that that kind of change is happening at the moment with regards to Docker style containers and how much you can rely on their isolation.

It's always been acknowledged that the larger attack surface of the Linux kernel led to a weaker level of isolation than things like dedicated security sandboxes or virtual machines, but the less quantifiable part is how much weaker is that isolation.

What's changing now is the ease with which an attacker can create container breakout exploits using LLM based tooling based on vulnerabilities found using other LLM based tooling. In the past the art of exploit creation was a fairly niche one and it took time and effort from a skilled professional to create a container breakout, which limited their use. However that's no longer the case.

## The case of CIFSwitch (CVE-2026-46243)

To provide a concrete example, I came across [this blog post](https://heyitsas.im/posts/cifswitch/) on a new Linux local privilege escalation vulnerability at the end of the working day, while browsing social media feeds. It's a great technical explanation of a new Linux LPE vulnerability which has recently been patched. Along with the technical blog they released a proof of concept which worked to escalate a normal users rights to root on a host.

Reading the blog, it looked like the kind of thing that might be usable as a container breakout, but I wasn't too sure if that'd work, and my C skills aren't really up to the task of finding out! In years gone by I would likely have kept an eye out to see if anyone created a breakout PoC for this, but otherwise not paid it that much more attention.

Now however, I could easily find out whether this is going to be exploitable by passing the information to an LLM and asking! 

There's some important nuance here of course. In order to get a good result there's a couple of pre-requisites. Firstly you need a strong model that's not going to object to creating proof of concept exploits. My favourite model for this is Anthropic's Opus 4.6. Later Opus models are quite strict on security work, so are unlikely to do this, but I've not found any offensive security related task that 4.6 won't happily undertake, which is nice.

The second pre-requisite is a validation loop that the model can use to actually create and test the PoC. Without that you're very likely to get a hallucination about whether this can be done and how, but asking for actual tested code avoids that problem. For this I use my [baremetalvmm](https://github.com/raesene/baremetalvmm) tool which is a piece of [personal software](https://raesene.github.io/blog/2026/05/10/personal-software-and-baremetalvmm/) that's well adapted for the task. It creates firecracker backed VMs which can have a custom kernel and rootfs allowing for speed of creation and easy customization.

With those two things in place, I gave Claude code a simple single prompt asking it to investigate this vulnerability using the blog post and existing PoC for reference and see if it could create a container breakout, I then went off to do other things and let the model run. 2 hours and $13 in tokens later, I had a working container breakout (available [here](https://github.com/raesene/vuln_pocs/tree/main/CVE-2026-46243)). The techniques it uses (including the PID spray) are all just a process of the input LPE PoC and the model's iteration.

## Wider implications

The combination of [many](https://xint.io/blog/copy-fail-linux-distributions), [many](https://github.com/V4bel/dirtyfrag),  [many](https://github.com/v12-security/pocs/tree/main/fragnesia), [many](https://github.com/0xdeadbeefnetwork/ssh-keysign-pwn) recent LPEs and the kind of ease of exploit creation we just described means it's sensible to re-evaluate the suitability of standard container isolation.

Personally my opinion is now that if you're using untrusted container images, or there's a risk that an attacker could execute code inside a container, you can't rely on that container for isolation at all, it should be assumed that the attacker can break out to the underlying host.

That doesn't mean that no-one should use containers, just that it's a good time to consider what uses you have for them and whether that matches up with your threat models and risk appetite.

