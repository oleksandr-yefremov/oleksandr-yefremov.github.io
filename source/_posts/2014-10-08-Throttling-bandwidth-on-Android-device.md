---
layout: post
title: "Throttling bandwidth for Android app on physical device"
date: 2014-10-08 14:41:15 +0300
comments: true
categories: utilities
published: true
---

The other day I needed to throttle (limit bandwidth) of video streaming application on Android for testing purposes. iPhone has this feature built in since iOS6.

<!-- more -->
As I use Mac OS, I will list instructions for it. Options that I considered:

**1. Use `emulator -netspeed gsm`**  
	I need to be able to do the same thing on real device as well, so strike off.
	
**2. Root phone, install or hack your own [application](https://play.google.com/store/apps/details?id=com.oxplot.bradybound&hl=en) which will limit bandwidth (or play with [iptables](http://stackoverflow.com/questions/4658619/how-to-use-iptables-in-an-android-application))**  
	Not an option, cause I need clean stock firmware on device.
	
**3. Throttle bandwidth on WiFi router, if it supports such a feature.**  
	And everybody connected to the same router will suffer. Besides, with some models you'd need to restart router every time you want to turn on/off throttling.
	
**3. If you have laptop (or desktop with WiFi interface), connect to network via Ethernet, setup Internet Sharing from Ethernet to WiFi, connect device to shared WiFi and use [Network Link Conditioner](http://nshipster.com/network-link-conditioner/) on Mac or similar software which is easy and straightforward but can only limit all connections together.**  
	This is the easiest way, if you don't mind all your connections on both laptop and smartphone slowed down. **NLC** also able to simulate packet drops and latency.
	
**4. Charles proxy**  
Lets you throttle specific host accessed with HTTP(S). You can also set breakpoints on everything.
Keep in mind that free version shows "waiting popups" occasionally and restarts every 30 minutes.

<a href="/images/throttling/Charles proxy throttling.png"><img class="caption" style="background-color:white;" src="/images/throttling/Charles proxy throttling.png" width="267" height="250 " /></a>

Since I might need to use RTP (or smth not HTTP) in future, let's take one more step.

**5. IceFloor or managing system firewall**  
"[IceFloor](http://www.hanynet.com/icefloor/) is a graphic frontend for [PF](http://en.wikipedia.org/wiki/PF_%28firewall%29)".

First we go through step 3 above: connect Ethernet, turn on Internet Sharing, connect our device to shared WiFi. Make sure it works, browse some page or stream video from your app. 
In **IceFloor** navigate to *Interfaces* tab, click `Update` button and locate Ethernet and bridge interfaces. In my case it was `en0`, `bridge100`.

<a href="/images/throttling/IceFloor Interfaces.png"><img class="caption" style="background-color:white;" src="/images/throttling/IceFloor Interfaces.png" width="374" height="214 " /></a>

Now to *NAT* tab, check "Share internet connection" and select interfaces that you located on previous screen.

<a href="/images/throttling/IceFloor NAT.png"><img class="caption" style="background-color:white;" src="/images/throttling/IceFloor NAT.png" width="374" height="214 " /></a>

Last step, *Firewall* tab, select "Outbound (NAT clients)" list, add required protocols to "Services in selected Address Group" (I just added "All services"). And here is the thing we've been chasing - Max. Bandwidth. Note, that we can't specify packet drop rate or latency here. If you need those as well as bandwidth control, **NLC** in step 3 is what I would use (or if you use HTTP as a protocol, **Charles** also can do this).

<a href="/images/throttling/IceFloor Firewall.png"><img class="caption" style="background-color:white;" src="/images/throttling/IceFloor Firewall.png" width="374" height="214 " /></a>

Click "Start PF" and you are good to go. Hit "Apply" every time you change any setting. To see if limit works for me, I usually go to [http://speedtest.net](http://speedtest.net).

And we can continue using **Charles** for debugging with **IceFloor** turned on.

P.S. Also I just noticed, that Internet Sharing allows me to see [preview](http://octopress.org/docs/blogging/) of Octopress post on mobile device as well as in desktop browser before I actually deploy it ;)



