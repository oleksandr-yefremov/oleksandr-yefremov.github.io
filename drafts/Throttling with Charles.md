---
layout: post
title: "Throttling your Android app with Charles Proxy."
date: 2014-10-08 14:41:15 +0300
comments: false
categories: tools and utilities
published: false
---

+ debug connection at the same time
+ does not require root

- http(s) only


The other day I needed to throttle (limit bandwidth) of video streaming application on Android for testing purposes. iPhone has this feature since iOS6.

Options that I considered:

<b>1. Use `emulator -netspeed gsm`</b>  
	I need to be able to do the same thing on real device as well, so strike off.
	
<b>2. Root phone, install application which will limit bandwidth (or play with `iptables`)</b>  
	Not an option, cause I need clean stock firmware on device
	
<b>3. Throttle bandwidth on WiFi router, if it supports such a feature.</b>  
	And everybody connected to the same router will suffer. Besides, with some models you'd need to restart router every time you want to turn on/off throttling.
	
<b>3. If you have laptop, connect to network via Ethernet, setup Internet Sharing through WiFi, connect device to laptop and use `Network Link Conditioner` on Mac or similar software which is easy and straightforward but can only limit all connections together.</b>  
	This is the easiest way  