---
layout: post
categories : 
- docker
title: "Running an X server inside a Docker container"
fulltitle: "Running an X server inside a Docker container"
excerpt: ""
---

Do you want an X server running in your Docker container? No X forwarding, no sharing sockets, no VNC/xrdp/xpra/NX/whatever. Just a proper X server running inside a container.

There is some information [on running X in LXC](https://mraw.org/blog/2011/04/05/Running_X_from_LXC/), [a SO question](http://stackoverflow.com/questions/26075741/starting-xserver-in-docker-ubuntu-container) and [lots of info in this thread](https://lists.linuxcontainers.org/pipermail/lxc-users/2010-August/000756.html), but do we have an up to date solution for Docker?

## A word to the wise

Running the X server needs privileged access to lots of hardware, which isn't good for isolation. But hey, containers are useful for loads of stuff and there are much better ways to sandbox applications.

## Caveats

- The udev-backed xinput hotplugging does not work yet so we have to define the mice and keyboards by hand.

## Step by step

Start a container, I am using Debian in this example because it is a relatively small base image.

<pre>
$ docker run -it --privileged debian /bin/bash

$ apt-get update

$ apt-get install xserver-xorg xorg jwm
</pre>

We installed the package for the X server and also [Joe's Window Manager](http://joewing.net/projects/jwm/).

That's pretty much it, because we started the container with `--privileged` you can just run `startx`. Make sure that you don't have an X server already running on your display, or arrange for the containerised X to use a different display number and VT.

You will notice however that the mouse does not move and the keyboard doesn't type. The xinput hotplugging does not appear to work, presumably because [udev is not supported inside a container](http://lists.freedesktop.org/archives/systemd-devel/2013-July/012347.html). The input devices can be configured manually in the X config.

### The X config

<pre>
Section "ServerFlags"
	Option "AutoAddDevices" "false"
EndSection

Section "InputDevice"
	Identifier "Keyboard0"
	Option "CoreKeyboard"
	Option "Device" "/dev/input/event2"
	Driver "evdev"
EndSection

Section "InputDevice"
	Identifier "Mouse0"
	Option "CorePointer"
	Driver "mouse"
	Option "Protocol" "ExplorerPS/2"
	Option "Device" "/dev/input/mice"
	Option "ZAxisMapping" "4 5"
EndSection
</pre>

Save this file inside the container as `/usr/share/X11/xorg.conf.d/10-inputs.conf` (on Debian/Ubuntu, at least). Whilst you are there, you might as well delete `10-evdev.conf` from the same directory.

This config tells the X server to not automatically add devices. The two `InputDevice` sections then define the keyboard and mouse. My keyboard device was `/dev/input/event2`, you might need to tweak this if this isn't where your keyboard is at.

# How useful is it?

In terms of security and sandboxing, running an X server in a container like this is practically equivalent to not running it in a container.

However, I find this interesting if you think of Docker more like a package manager. You could mix a cutting edge X server into a more stable version of your favourite distro, or mix an older X server into a newer distro if you need some legacy driver that is no longer available.

I did exactly that, mixing the X server from Ubuntu 14.04 (the latest version that Matrox is supporting with the M91xx drivers) into Ubuntu 15.10.
