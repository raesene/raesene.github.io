---
layout: post
title: "Changing Times - End of SMS Auth?"
date: 2014-11-02 21:10:58 +0000
comments: true
categories: Authentication
---
So like most things in security, decisions are made based on a set of assumptions, and it's when these assumptions prove to be faulty that security can go quite badly wrong.

I was thinking about a fairly common assumption recently in light of Apple's new [continuity](http://support.apple.com/en-us/HT6337) feature.  Over the last couple of years as the weaknesses in password based authentication models become ever more apparent, there's obviously been a drive to improve authentication, and one method of doing this is through the use of different channels to send authentication tokens "out of band".  

One of the more common methods of doing this is to send authentication tokens over SMS messages.  The theory is that this is a separate channel from the PC or tablet that is the primary device being used, so it should be harder for an attacker to compromise both channels, thereby improving security.

For example if we have an attacker who has access to a users PC (for example having placed malware on it which steals passwords and other data entered there), the idea goes, by adding in SMS, we make it harder for the attacker to compromise the whole authentication mechanism.

You can probably see, based on the starting paragraph, where this assumption is starting to fall down... With features like Continuity or, SMS to email applications or phone carriers who provide websites which allow for viewing and sending SMS messages, the barrier between these two channels is effectively pierced and SMS becomes much less useful as a second factor for authentication.

Now if an attacker can compromise the PC, it's quite likely that they'll get access to the users SMS messages at the same time, with all that entails.

Hopefully this line of thinking will penetrate with service operators and they'll realise that if 2-factor/channel authentication is needed for their service, they'll need to offer something more robustly separate.

The other thing I thought this was an interesting illustration of, is that this to me is a good example of differing incentives in the security world.  Online service provides have the incentive for SMS messaging to be a good 2nd channel for authentication data as it's something that almost all their subscribers already have access to, and it's very cheap to use.

On the other hand, mobile operators and mobile ecosystem providers have no such incentive.  Instead they're working to make it easier for users to access all their information in a single place....