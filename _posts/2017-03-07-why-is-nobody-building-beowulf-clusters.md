---
layout: post
comments: true
title: "Why is nobody building beowulf clusters?... because Linux on desktop PCs is s**t"
fulltitle: "Why is nobody building beowulf clusters?... because Linux on desktop PCs is s**t"
excerpt: ""
categories : 
- notes
- rants
- hpc
---

> This post is a rant

I've used Linux almost exclusively for a long time and had the impression that it is fairly stable and hardware support is good. But when I think about the kind of hardware I've ran, it has been generally _Linux friendly_: most of the T-series ThinkPads for example. Forgive the odd bit of non-free firmware for stuff like network cards, it's plain sailing.

Notably, I don't really use a desktop PC at all.

At the university we maintain a _beowulf cluster_ built from commodity hardware (usually PCs from labs when they are refreshed) and it's good for teaching students about parallel and distributed computing. Naturally, it is a few generations behind in hardware.

So as we leave the _Core 2 Quad Q8300_ era and usher in the new _i5 2310_ systems, here is a non-exhaustive list of issues we encountered:

-----

## buggy BIOS implementations

Disable all power management options, C-states, and don't forget to try those options that are not even present on the desktop boards? Sure thing.

## graphics drivers that crash

Reboot every day when the display freezes? No problem.

## network cards that only work with __all__ offloading disabled

Have the CPU perform all network functions? Why not.

-----

These solutions might be okay for casual users, but for a cluster it makes the hardware unusable.

These "new" systems are a few years old now... hardware support should be good for this kit, but every bug is marked as WONTFIX, offering silly workarounds (as above) and met with the question _"Why are you using old stuff?"_.

Part of the blame is on the vendors like **Stone** and **Viglen** for producing bad hardware, but the driver support for these common as muck Intel chipsets is bad too.

Those with the knowledge to fix it have no desire, and it's not for a lack of volunteers with hardware and a willingness to test. Maybe we must accept that this is the Linux way.
