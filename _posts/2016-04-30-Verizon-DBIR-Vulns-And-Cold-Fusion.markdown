---
layout: post
title: "Verizon DBIR, Vulnerabilities and Cold Fusion"
date: 2016-04-30 07:50:39 +0100
comments: false
categories: Vulnerabilities
---

So it's Verizon DBIR time of year again and as with last year there seems to be a little bit of debate around the Top 10 exploited CVEs.  My twitter handle got copied in via some tweets from last year, so I thought I'd take the opportunity of providing a tester's perspective on this. A more detailed and comprehensive look at this issue is available on the [OSVDB Blog](https://blog.osvdb.org/2016/04/27/a-note-on-the-verizon-dbir-2016-vulnerabilities-claims/).

The thing that sprung out to me from the list was the first two mentioned as top 10 successfully exploited issues on the OSVDB blog.

 * 2015-03-05 – CVE-2015-1637 – Microsoft Windows Secure Channel (Schannel) RSA Temporary Key Handling EXPORT_RSA Ciphers Downgrade MitM (FREAK)
 * 2015-01-06 – CVE-2015-0204 – OpenSSL RSA Temporary Key Handling EXPORT_RSA Ciphers Downgrade MitM (FREAK) 

These CVEs related to the [TLS FREAK attack](https://freakattack.com/) which is an SSL cipher downgrade issue.  They jumped out at me as I see these on tests quite a bit and frankly, they don't generally get rated as that serious of an issue.

To draw an analogy if it turns out that these were two of the most successful attacks of 2015, this is kind of like (to me) a scientist saying they've perfected **cold fusion**! a) it's very unlikely and b) if true it's extremely significant and deserves a lot more prominence than a footnote in the DBIR!

## Why is it unlikely?

It's unlikely because to exploit this issues there's a number of pre-requisites that need to be satisfied. 

1. **MiTM** - This is the big one.  This issue is only exploitable when the attacker can sit in a position to intercept and modify traffic between the client and the server.
2. **Client + Server Vuln. required** - Unlike some of the other SSL vulnerabilities we've seen this requires a vulnerable client and server. Most popular browsers got patched for this pretty quickly (either via auto-update or monthly patch) so the window of attack for most users was pretty small.
3. **Cryptographic Power**.  We're talking about the ability to mass crack ciphers here.  Now they're not strong ones (that's the point of the attack) but still this is some endevour to do at scale.
4. **Detecting Exploitation**.  To detect exploitation requires that the sensor can tell when the ciphersuite is downgraded,  the attacker cracks the key and then presumably carries out some follow-on attack.


## What does it mean?.

So lets assume these are indeed two of the most exploited vulnerabilities of 2015, what would that mean?

 * There are a set of attackers who have MiTM positions to attack large numbers of clients and servers on the Internet
 * Those attackers have the processing power to mass-crack cryptographic keys gathered from intercepted connections presumably quickly enough to do something with the key in question.

If this is true that's a pretty serious problem, as this set of attackers are hardly going to stop what they're doing once the patch rate for FREAK improves as it did quite rapidly.

Anyway, from the OSVDB blog I understand that Kenna are planning a explanatory blog posting, which I very much look forward to reading.  Things I would really like to see in that would be 

 * Details of how detection successful exploitation of FREAK was achieved.
 * Details on what the attack group did with this afterwards.
 * Details of what threat group carried out these attacks. 