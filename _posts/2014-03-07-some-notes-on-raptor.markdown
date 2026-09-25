---
layout: post
title: "Some Notes on Raptor"
date: 2014-03-07 19:48:24 +0000
comments: true
categories: [Web App Testing, Product Notes]
published: false
---
Here's something I always mean to do and very rarely do, make notes on products as I find them.  So I've been testing an EZProxy installation recently and I noticed that it comes with a product called Raptor running on port 8112/TCP which is used for accounting/reporting and the like.

First point is that default installations have default credentials *sigh*.  Logging in with admin/raptor may provide access.

Also directory indexing appears to be enabled on the web server by default.

This can lead to interesting directories like /download/ which can contain reports