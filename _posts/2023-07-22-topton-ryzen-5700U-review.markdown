---
layout: post
title: "Hardware review - Topton AMD Ryzen 7 5700U" 
date: 2023-07-22 10:27:00 +0100
comments: false
categories: Hardware
---

I've been thinking about a project where I could travel to conferences with hardware to run a workshop that doesn't rely on cloud resources (to avoid those concerns about network access or cloud uptime) and of course, the first part of any project like that is the fun part, buying the hardware! As I'd promised a review to a couple of people on-line, here it is.

The goals for the hardware were to have something small and light with enough resources that it could support 10-20 students running on it, without breaking the bank. So, I decided to try getting a mini PC from AliExpress, as they have some pretty good deals on there.

After a bit of looking around and asking friends who had bought systems like this from AliExpress before, I settled on a [Topton AMD Ryzen 7 5700U](https://www.aliexpress.com/item/1005005762253642.html). The model I got cost £352.53.

The purchase and shipping process was relatively smooth. After clarifying my address with the seller it took 8 days for the system to arrive. It was well packaged and arrived in good condition.

## Specs

- CPU - AMD Ryzen 7 5700U with Radeon Graphics
- RAM - 64GB DDR4 3200MHz, fitted as 2x32GB SODIMM
- Storage - Fanxiang S500PRO 1TB
- Network - `RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller`
- Wi-Fi - Wifi6

## Physical overview

The system is pretty small, measuring 11.5cm x 10.5cm x 4cm, it comes with a power supply and an HDMI cable. The power supply output is a rated as 19V and 3.42A, so 65W. Physically it's fairly small, 10.5cm x 4.5cm x 3cm. The case is metal and feels fairly solid

![Topton machine from the top]({{site.url }}/assets/media/shoppy.png)

The port selection is pretty good for such a small machine. On the back there's power, 2xHDMI, Ethernet, 2xUSB-A, 1xUSB-C and a headphone jack, and on the front another 2xUSB-A.

![Topton machine back ports]({{site.url }}/assets/media/shoppy-ports.png)

## Noise

Anecdotally I found the system to be relatively quiet. I've not got a good decibel meter, but took some measurements just using an iPhone app. when the host is idling at 5cm distance from the side, the reading was 30dB, which is around the standard background noise where I am. Periodically with nothing much running there's a bit of fan noise, taking it up to ~41dB.

Running a stress test with `stress-ng` on 2 cores, to cause the fans to spin up, takes the volume at 5cm to ~51dB. So it's not silent, but it's not too loud either.

## Software

The system came with an install of Windows 11 on it (not sure of the licensing situation for that install...), but for my purposes I wanted Linux, so I installed Debian 12 (Bookworm) on it. The installation process was fine, and the only thing I can see which isn't supported (yet) is the Wi-Fi which is a Realtek device. A bit of searching suggests that'll get a drive in the 6.2 kernel.

## Performance

Next question of course is, what's the performance like? For this I used the [phoronix-test-suite](https://www.phoronix-test-suite.com/) to run a few benchmarks using their docker image `docker run -it phoronix/pts`, with results uploaded to [OpenBenchmarking.org](https://openbenchmarking.org/). 

- [Apache](https://openbenchmarking.org/result/2307223-NE-APACHERES42)
- [iozone](https://openbenchmarking.org/result/2307228-NE-SHOPPYIOZ32)

Overall the numbers seem pretty reasonable for a system of this class.

## Conclusion

Overall this seems like a pretty good system for the price. Next step will be looking at how to build it as a workshop host for container security workshops.