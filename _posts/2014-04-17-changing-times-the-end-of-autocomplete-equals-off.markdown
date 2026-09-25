---
layout: post
title: "Changing Times - The end of Autocomplete='off'"
date: 2014-04-17 15:08:36 +0100
comments: true
categories: [Browsers, Testing]
---

For a long time the subject of browser password storage has been a relatively contentious topic in the Information Security world.

Whether security is improved by browsers being the custodians of users passwords is a potentially debatable point.

On the one hand having browsers manage it, could free users to choose strong unique passwords for every site they use (whether they do or not is another matter).

On the other hand users may not be conditioned to realising that if you do this, you need to be extremely careful who you let use your PC!

The actual list of arguments and risks pro and con is pretty long, and I'm not going to try and list them all for the inevitable reason that I don't know 'em all :)

Essentially I'd say that in the majority of cases a well designed browser password manager is likely to improve overall security, but there are cases where the risk model/use case of a site might predicate against that.

Either way in the past the way of resolving this has been for individual sites to set the autocomplete="off" flag on password fields or login forms of their sites.  It's a common recommendations from security reviewers and auditors as well who view passwords being stored on potentially malware infected/untrusted systems as a bad thing.

However in the last year or so browser makers have decided that this choice is harmful to user experience/security and have decided to remove it.  With [Firefox making the change](https://bugzilla.mozilla.org/show_bug.cgi?id=956906) over to disable autocomplete="off" by default on all password fields, that's the last of the major PC browsers to move over to that stance, as Chrome made the [change with version 34](http://www.theregister.co.uk/2014/04/09/chrome_makes_new_password_grab_in_version_34/) and Internet Explorer [changed it in IE 11](http://msdn.microsoft.com/en-us/library/ms533486%28VS.85%29.aspx).

A side effect of this is that sites may as well remove any "remember me" style functionality that they had as this is essentially redundant, the browser will let the user decide whether they should cache their credentials for that site and how long for (well essentially forever).

Whilst this is interesting in and of itself (especially to see how long it takes security reviewers and standards bodies to update their guidance), perhaps a bigger point is that for companies who have standardised their software on browser platforms (as most seem to these days), it's a reminder that they are not in complete control of the user experience, and that if the browser makers decide to change something, they may have limited abilities to stop that change, whether they like it or not.