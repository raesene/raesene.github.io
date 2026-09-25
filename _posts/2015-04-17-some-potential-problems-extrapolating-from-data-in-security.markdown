---
layout: post
title: "Some potential problems extrapolating from data in security"
date: 2015-04-17 18:08:34 +0100
comments: true
categories: Statistics
---
One of the perennial problems in security is the lack of hard data, so it's been good to see over the last couple of years a growing number of reports coming out which seek to shed a bit more light on what's happening in InfoSec.  One of the more prominent of these reports is the [Verizon Data Breach Investigation report](http://www.verizonenterprise.com/DBIR/2015/) and I always read it as it has some interesting insights into what's happening in areas that I don't get too much exposure too.

However when I read this years report I noticed a section which looked .. well odd.  It's on page 16 of the report and talks about the top 10 "exploited" vulnerabilities in 2014.  What struck me as very odd is the number of very old vulnerabilities in the list.  Eight of the ten issues listed are from 2002 or before.  Now I know IT security patching can be a bit slow, but this seems excessive!

So I had a look at the CVE references provided and what I found did not ehance my confidence that this data was an accurate reflection of exploited issues.  In a number of cases the referenced CVEs are for old and or obscure products that would surprise me if they're in wide deployment.

The best example of this problem is [CVE-2002-1931](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-1931) which gets listed at number nine.  This is a Cross-Site Scripting issue in version 2.1.1 and 1.1.3 of a product called PHP Arena and specifically the pafileDB area of that product.  Now I struggled to find out too much about that product because the site that used to host it www.phparena.net is now a gambling site (I presume that the domain name lapsed and was picked up as it got decent traffic).  Searching via google for information, most of the results seemed to be from vulnerability databases(!) and using a google dork of [inurl:pafiledb](https://www.google.co.uk/?gws_rd=ssl#q=inurl:pafiledb.php&start=170) shows a total of 156 results, which seems low for one of the most exploited issues on the Internet.

Also looking at [this securityfocus page](http://www.securityfocus.com/bid/6021) we can see that PHP Arena pafileDB 3.0 isn't vulnerable to this issue and from [this cvedetails page](http://www.cvedetails.com/product/2451/Php-Arena-Pafiledb.html?vendor_id=1413) we can see that pafileDB 3.0 came out in 2002.

So for this issue to be in the top 10 most exploited issues on the Internet, we need a large number of people to not have updated a very exploitable, discontinued PHP forum product for 13 years...

When I saw this initially, I thought "ahh must be a problem with the publishing", so I contacted Verizon by e-mail and then [over twitter](https://twitter.com/raesene/status/589022276252782592), however apparently not, this is the real data!  They were very nice and provided some additional detail (although it's hard to be specific in 140 chars) which indicates that this data came from IDS sensors correlated to vulnerabilities, but I can't really see how that could possibly be the case.  

If it were just raw network IDS sensor data it's possible if it were just looking at network traffic and alerting on vulnerability scanners testing for things, but a) that's not exploitation of an issue, just scaning for its presence and b) even then things like the PHP arena one don't make sense unless the IDS sensor is just mistakenly putting all XSS attempts into that bucket.  Either way I'm afraid this looks like mistaken interpretation of the available data and an extrapolation based on that interpretation

So why do I care enough to write this?  Well unfortunately what a lot of people in the industry will do here is take the headline "97% of exploited issues came from these 10 issues" and use this to drive spending and other decisions, and that's unfortunate as without looking at the detail you don't realise that there are problems with that interpretation of it.

For completeness here's some notes on the other issues.  The first two are generic SNMPv1 issues from 2002 which from their descriptions [CVE−2002−0012](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-0012) and [CVE−2002−0013 ](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-0013) look kind of like the same thing and also are not very specific as it's multiple vendors.

Number three is default SNMP community strings [CVE-1999-0517](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-1999-0517) which is kind of believable as you do see a fair bit of SNMP out there, although in my recent experience default community strings on the Internet are rarer.

Number four is a memory leak DoS in Windows NT 4 and 2000 [CVE-2001-0540](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2001-0540) RDP requests. Now both these products have been out of support for some years and are less seen these days, also with a DoS issue whilst it's possible to detect someone attacking it with network IDS, I don't see how you'd easily detect actual exploitation without something running on the target server to note the service crashing.

Number five is a newer one, [POODLE](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566).  This being in the top 10 exploited issues would be very interesting as traditionally SSL issues which require an active Man-In-The-Middle attack are considered less exploited (usually 'cause if the person has a privileged position on the network like that, they can do far worse things that exploit SSL issues!).  Now I'd believe it was one of the top "scanned for " issues but not exploited.

Number six is [another RDP DoS](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2012-0152) .  This one is more modern, but the problem of accurately detecting DoS exploitation remains..

Number seven is a pretty obscure one [Directory traversal vulnerability in ftpd in QPC QVT/Net 4.0 and AVT/Term 5.0](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2001-0680).  As with the PHP Arena, there's not a lot of information on the Internet about this product.  the host company still exists but no download/version information seems easily available.  That said from [Securityfocus](http://www.securityfocus.com/bid/2618) the latest affected OS is Windows 2000 SP2, which hasn't been in support for a long time and has other critical exploitable issues, so I'd have guessed that if someone was going to exploit it they wouldn't have targeted this issue...

Number eight is a directory traversal issue in [Pablo FTP v1.0 build 9](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-1054) . From a quick search the product was up to [version 3.2](http://www.pablosoftwaresolutions.com/html/quick__n_easy_ftp_server.html) when it was last released and that version supports up to Windows vista, so it's a fair guess that version 1 hasn't been in use/supported for a while.

Number nine is the PHP Arena issue mentioned previously.

Number ten is a bug in [windows 2000 and XP](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-1932) where administrators aren't notified if logs fill up. If a system isn't patched for this they also have all the canonical remote code execution bugs like MS06-040 and MS08-067.  So for this to be more exploited, we have to have large numbers of attackers who want to fill up logs with impunity, instead of taking over the system.