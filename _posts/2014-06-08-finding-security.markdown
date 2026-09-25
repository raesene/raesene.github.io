---
layout: post
title: "Finding security"
date: 2014-06-08 19:11:23 +0100
comments: true
categories: Ramblings 
---

So how do we know that the sites we use are secure?  It's an obvious question with no brilliant answer but here's an example of how we don't.

What's below might well come off as getting at Kickstarter, but really it's not that different from a wide number of other sites that would like us to entrust them with personal information or credit card details.

****

I'm a fan of [Kickstarter](http://www.kickstarter.com) projects, I've backed quite a few and they've gone pretty well for the most part.  Until recently whenever I went to contribute to a Kickstarter I felt reasonably happy paying, as they used Amazon payments, which meant I just logged in to Amazon and authorised the payment, all good.  

I'm not a fan of giving sites my credit card details these days, so if I don't know you and you don't offer Amazon Payments/Paypal I probably won't buy from you, sorry but having my card re-issued is a right pain.

So when I went to Kickstarter to support the [Scandanavia and the World webcomic](https://www.kickstarter.com/projects/1438393276/scandinavia-and-the-world-animated) that I rather like, I was a bit put out to find that now the site has a form asking me to enter my card details or use a stored card.

![screenshot](http://imgur.com/a6lLh80.png)

**Yeuch** they're asking for my card number and CVV, what's more there's a little graphic secure padlock which has no actual effect, always makes me nervous when I see those.

So I thought "well I'm going to do a bit of looking about to see if I can get a good feeling about their security before I entrust them with my card"

I looked at the source of the page to see if they perhaps POST to a payment processor, it's not great practice but hey it's better than nothing if it's one who I think will be ok.  

From what I could see nope is the answer looks like they POST locally.  Now they may be doing something fancy in JavaScript before posting (like stripe does) but looking at the JavaScript on the page, there's reams of minified code that I'm not going to try understanding and I don't fancy starting putting Proxies in-line and entering dummy numbers to see exactly what's happening.

OK so maybe they have a security policy with some good content on how they secure my data .. well if they do I couldn't find it, there's a privacy policy that doesn't mention the word security at all.

Then I looked through the support section and did find this under payments > Security

> All transactions made through Kickstarter on our secure server are safe. A software protocol called Secure Sockets Layer Secure (SSL) is used for all pledges. It encrypts all of your personal information--including credit card number, name and address- as it travels over the internet. SSL is the industry standard and among the best software available today for secure commerce transactions. All of the credit card information is stored on a secure, dedicated database.

What does this tell me.  Well not a lot really apart from that they use SSL and that everything is fine...

Actually the closest thing I found to good information on the topic was in the blog posting about their [February 2014 breach](https://www.kickstarter.com/blog/important-kickstarter-security-notice) which mentions that they use bcrypt for password hashing, which is a good thing.  Apart from that it's the same vague positive stuff with no actual details as is in the support site.

Failing to find anything directly related to security, I thought I'd have a quick look for some other signs that a company take these things seriously

 - Dedicated e-mail address for security displayed prominently on the contact details - **Nope** (there is one on the blog posting but it's not really clear if that's just for questions on the breach or more general security stuff)
 - Mentions of specific staff members with responsibility for security on the team page - **Nope**
 - Bug bounty program to let security researchers responsibly disclose issues with the site - **Nope**
 
 Where does all this leave me.  Well in this case I'll probably go and dust off one of the virtual credit cards I've used in the past and use that for Kickstarters in the future, but it is a shame that in 2014 this is still standard practice with an awful lot of companies.
