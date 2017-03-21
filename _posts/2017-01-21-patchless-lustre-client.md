---
layout: post
comments: true
title: "Patchless Lustre client on new kernel versions"
fulltitle: "Patchless Lustre client on new kernel versions"
excerpt: ""
categories : 
- hpc
- lustre
- howto
---

Lustre is one of the most popular parallel filesystems.

Depending on if you are setting up a Lustre server, router or client, a kernel patch may or may not be required. In any case, you will probably want to build the client modules to mount the filesystem ;)

If you are running a RedHat derivative (and the distribution provided kernel) this is usually no problem, because there are RPMs for everything.

However, there is very little documentation on running Lustre outside of the compatibility matrix, even though it has had DKMS support, and has been moving to the mainline kernel, for some time.

For this reason we took the conservative approach of sticking with the old 3.10 kernel on CentOS 7, and the tried and tested Lustre 2.8 packages - 2.9 is now available.

For the servers and routers this is fine.

But it is frustrating for me on the client, where you actually want to access the filesystem from, because of two main reasons:

- other software that requires kernel patches, for which you cannot find a common kernel version that both it and Lustre are happy with
- other distributions, like Ubuntu, that are not well supported

To cut a long story short...

## Lustre client modules are in staging

You can find out if you have it already. For example, on Ubuntu 15.10:

```
root@joshiggins-Q87M-E:/home/joshiggins# uname -r
4.2.0-34-generic
root@joshiggins-Q87M-E:/home/joshiggins# modprobe lustre
root@joshiggins-Q87M-E:/home/joshiggins# dmesg | tail
[  597.286225] Lustre: Lustre: Build Version: v2_3_64_0-g6e62c21-CHANGED-3.9.0
[  597.313586] ptlrpc: module is from the staging directory, the quality is unknown, you have been warned.
[  597.325970] ksocklnd: module is from the staging directory, the quality is unknown, you have been warned.
[  597.326958] LNet: Added LNI 172.17.4.16@tcp [8/256/0/180]
[  597.326983] LNet: Accept secure, port 988
[  597.326993] LNet: 3300:0:(lib-eq.c:85:LNetEQAlloc()) EQ callback is guaranteed to get every event, do you still want to set eqcount 1 for polling event which will have locking overhead? Please contact with developer to confirm
[  597.336942] lov: module is from the staging directory, the quality is unknown, you have been warned.
[  597.350144] fid: module is from the staging directory, the quality is unknown, you have been warned.
[  597.352555] mdc: module is from the staging directory, the quality is unknown, you have been warned.
[  597.359797] lustre: module is from the staging directory, the quality is unknown, you have been warned.
root@joshiggins-Q87M-E:/home/joshiggins# 
```

As you can see it looks like we have a Lustre module version 2.3. All you need are the Lustre utils (`mount.lustre`, `lctl`, etc) in order to use it.

## Compile Lustre utils from git

Clone the Lustre git repository, I used [rread's github mirror](https://github.com/rread/lustre.git).

There are branches for the different versions, so make sure you checkout the most appropriate for your version.

Configure just for the utils:

```
./configure --disable-server --disable-modules --disable-client --disable-tests --disable-manpages
```

A quick `make`, `make install`, and we have the utilities installed.

```
root@joshiggins-Q87M-E:~# lctl list_nids
172.17.4.16@tcp
```

## Finally

I was able to mount the Lustre filesystem on this Ubuntu machine.

```
root@joshiggins-Q87M-E:~# mount -t lustre 172.17.4.19@tcp:/scratch /scratch
root@joshiggins-Q87M-E:~# df -h | grep scratch
172.17.4.19@tcp:/scratch   19T  4.4T   13T  26% /scratch
```

Even though interoperability between Lustre versions is pretty good, I don't think it would be a good idea to run a production system with such a large difference in client and server versions.