---
layout: post
title: "Fixing DVD The Playing Issue on Linux Mint 20 Xfce"
categories: reos-linux
comments: true
---

Putting Linux on an old computer brings it back into service, when it otherwise would have ended up in landfill. If you happen to have an extensive collection of DVDs (or live near a library or op-shop that has one), old computers that have an internal DVD-ROM drive should easily become a nifty portable DVD player.

When I tried doing this on Linux Mint 20 Xfce however, the built-in DVD playing software [Celluloid](https://celluloid-player.github.io/){:target=”_blank”} gave an `unsupported format` error. I tried downloading venerable [VLC](https://www.videolan.org/)/){:target=”_blank”}, same thing. Hunting through the 'net, I found and compiled a list of utilities to make DVDs play on Linux Mint 20, mostly from [this post](https://forums.linuxmint.com/viewtopic.php?t=267440){:target=”_blank”}. 

So if you have had this same problem, you have come to the right place! Here for your convenience are the list commands that I ran, and soon after that I was watching DVDs on my ex-Windows XP laptop, no worries (both Celluloid and VLC worked):

```
sudo apt-get update
sudo apt-get install udftools
sudo apt-get install libdvdcss2
sudo apt-get install libdvdnav4
sudo apt-get install libdvdread4
sudo apt-get install mencoder
sudo apt-get install libdvd-pkg
sudo dpkg-reconfigure libdvd-pkg
```
