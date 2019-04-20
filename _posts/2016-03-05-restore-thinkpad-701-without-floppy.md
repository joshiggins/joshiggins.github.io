---
layout: post
comments: false
categories: 
    - howto
title: "Restore the ThinkPad 701 without a floppy drive"
fulltitle: "Restore the ThinkPad 701 without a floppy drive"
excerpt: ""
---

The excellent OS/2 Museum details how to restore a ThinkPad 701C using a CF card located in the PCMCIA slot and a bootable floppy.

Only problem is, I don't have a floppy drive for this machine and they seem to be hard to find nowadays.

The solution is to resort to cheating with modern technology, but that isn't a problem as I was planning to replace the old HDD with a CF card inside a IDE to CF adapter like this. It turns out the CF card I had wouldn't boot in the machine (probably not supporting ATA 33) but the method works just as well with the old HDD.

In a nutshell, the system can be prepared on another PC before putting the CF card or HDD into the ThinkPad.

The files used can be found on the link to the original article on OS/2 Museum.

## The process

Basically, we will use QEMU and Michal's adapted boot floppy to prepare a complete image to write to the CF card or HDD for the ThinkPad.

### 1. Create a blank image for the CF card

```
dd if=/dev/zero of=cf.img bs=450M count=1
```

This image will be written to the CF card at the end. I used a 512MB card so was conservative.

### 2. Boot the floppy image (1st time)

```
kvm -fda tp701rescf.img -hda cf.img -m 16 -boot a
```

You will see the PCDOS starting, and the first question of the boot floppy. Press 'Y' to continue and then press 'C' to format the "hard drive".

The system will reboot but the preload will fail because the zip files are still missing. The important thing is now we have an image file containing the correct partition and filesystem.

### 3. Create the image holding the preload zip

Now we will make a copy of the freshly formatted `cf.img` image, mount it, and copy the preload zip to it. This will essentially be equivalent to the PCMCIA/CF drive used in the original article.

```
cp cf.img zips.img
mkdir ztmp
mount -t msdos -o loop,offset=32256 zips.img ztmp
mkdir ztmp/zip
cp mobile2.zip ztmp/zip/mobile2.zip
umount ztmp
```

### 4. Boot the floppy image (2nd time)

This time we will run the floppy image with the `zips.img` as the second hard drive and the preload should procced.

```
kvm -fda tp701rescf.img -hda cf.img -hdb zips.img -m 16 -boot a 
```

<img src="/assets/images/QEMU_006.png" width="300px"/>

### 5. Write to the real device

After the preload is complete the `cf.img` should be ready to write to the real device, sdX.

```
dd if=cf.img of=/dev/sdX
```

## The result

An X32 was enlisted to write the image to the old HDD as I didn't have a laptop IDE drive adapter handy, and the thicker drive managed to squeeze in after removing the palmrest and hard drive slot assembly.

<img src="/assets/images/IMG_20160306_005448.jpg" width="300px"/>

After returning the drive to the 701, it boots right up into the hilarious ThinkPad Advisor, success!

<img src="/assets/images/IMG_20160306_012109.jpg" width="300px"/>
