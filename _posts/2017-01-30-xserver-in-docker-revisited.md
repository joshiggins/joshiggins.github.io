---
layout: post
comments: true
title: "X server inside Docker revisited, with input device hotplugging"
fulltitle: "X server inside Docker revisited, with input device hotplugging"
excerpt: ""
---

Last year I posted an article about [running an X server inside a Docker container](http://joshh.info/2016/xserver-inside-docker-container/). The caveat being that for some reason input device hotplugging did not work, and the *udev* devs on the mailing list said that it is not supported within Docker. At the time I left it at that.

Well I stumbled across [this short post by dummdida](https://dummdida.tumblr.com/post/121087781445/re-udev-events-in-a-container) that reveals it's possible to get udev events into the container. Should have guessed so really - if you are in the host's networking namespace, and you share the same `/dev` folder, why not?

I tried, it didn't work.

It turns out that neither of those things are required (`--net=host` and `-v /dev:/dev`)!

All we need is to start a *udevd* daemon in the container. After this, the input devices worked after being unplugged and re-plugged. To solve this last step, an easy `udevadm trigger`!

This raises some questions that I am not familiar enough with these things to answer;

- will running more than 1 *udevd* daemon (host's and container's) have any unintended side effects?
- why can't we use the host's *udevd* like dummdida described for Xorg?

But it is working, so without further ado...

## Step by step X server in Docker container, with input device hotplugging

```
$ docker run -it --privileged debian /bin/bash
$ apt-get update
$ apt-get install xserver-xorg xorg jwm
$ udevd --debug &
$ udevadm trigger
<lots of output, brace yourself>
$ startx
```

You should now have an X server running with a working keyboard and mouse, without any manual X configuration.

## Finishing up

The same warranty applies that comes with the previous article - generally running containers in privileged mode is not recommended from a security perspective.

I find this solution a really cool idea if you think of like a package manager and forget about any kind of security isolation, which in my opinion is a perfectly valid application of container technology - reproducibility.