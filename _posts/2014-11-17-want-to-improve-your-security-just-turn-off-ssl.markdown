---
layout: post
title: "Want to improve your security? Just turn off SSL!"
date: 2014-11-17 18:48:16 +0000
comments: true
categories: SSL
---

I've had cause to work with some of the more common Vulnerability Assessment scanners recently, and I couldn't help but notice that a lot of the findings related to incorrect configuration of, or bugs with, SSL implementations.  

A lot of network services use SSL to provide encryption and authentication of the server these days and default configurations a generally not ideal, with things like Self-Signed certificates being common.  In addition to that the quite large number of SSL bugs means that there can be quite a few findings relating to each SSL endpoint on a given host.

So far, so expected.  Vulnerability Assessment scanners are meant to flag up this kind of issue.  However what struck me about this is that mis-configured encryption seems to be a lot worse (from a findings point of view) than no encryption at all, which doesn't seem right at all.

Lets take the example of telnet.  In Nessus this seems to be a Low with a CVSS base score of [2.6](http://www.tenable.com/plugins/index.php?view=single&id=42263), Nexpose has this issue as a CVSS [4](http://www.rapid7.co.uk/db/vulnerabilities/telnet-open-port). (BTW I'm not particularly focusing on these scanners for any reason other than they're the ones I'm most familiar with).

In comparison to this having a self-signed certificate in Nessus is a [6.4](http://www.tenable.com/plugins/index.php?view=single&id=57582) and in Nexpose is [4](http://www.rapid7.co.uk/db/vulnerabilities/ssl-self-signed-certificate), or supporting SSLv2 which is a [5](http://www.tenable.com/plugins/index.php?view=single&id=20007) or [6](http://www.rapid7.co.uk/db/vulnerabilities/sslv2-and-up-enabled)

So on the one hand we have a completely unencrypted service which transmits credentials and data in the clear and is vulnerable to passive traffic sniffing and active MITM attacks. On the other hand we have vulnerabilities which may allow for an active MITM and wouldn't allow for passive sniffing, and yet the encrypted solution is rated as a more severe issue in Nessus and equally as bad or worse in Nexpose.

The only good argument I've heard in favour of this situation is that the encrypted services give a false sense of security.  That could be true, but in this case it's not relevant as CVSS base score don't account for that...

So why does this matter, surely any compentant tester will look at these scores, chuckle, and rate the findings appropraitely?  Sure, but in situations like PCI DSS ASV scanning, the CVSS base score is law, unless the customer wants to fight it point by point, so it's kind of important these things are logical.

What could be done about it?  Well for me, any weakness in SSL or other encrypted protocol should automatically be lower rated than a completely unencrypted protocol, for kind of obvious reasons.  

