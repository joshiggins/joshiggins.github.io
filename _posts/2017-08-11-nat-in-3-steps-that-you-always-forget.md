---
layout: post
comments: true
title: "NAT in 3 steps (that you always forget)"
fulltitle: ""
excerpt: ""
categories : 
- howto
---

In this guide, replace `<external>` and `<internal>` with the device names of your external and internal network interfaces respectively.

You can list your network devices with the `ip addr` or `ip link` command.

## 1. Allow IP forwarding

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

## 2. Enable masquerade

Masquerade allows all the hosts on the server's internal network to hide behind and use it's IP address on the external network.

```
iptables -t nat -A POSTROUTING -o <external> -j MASQUERADE
```

## 3. Set forwarding policy

Two way NAT (private hosts can get out and external hosts can get in)

```
iptables -A FORWARD -i <internal> -o <external> -j ACCEPT
iptables -A FORWARD -i <external> -o <internal> -j ACCEPT
```

One way NAT (private hosts can get out, only connections they established can come back in)

```
iptables -A FORWARD -i <internal> -o <external> -j ACCEPT
iptables -A FORWARD -i <external> -o <internal> -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## (Optional) Make permanent

You can edit `/etc/sysctl.conf` adding the line

```
net.ipv4.ip_forward = 1
```

to always allow IP forwarding when the system reboots.

Then, create a script to apply your IP tables rules. Most distributions allow you to put commands in `/etc/rc.local` that will be executed on boot. Some distributions have their own methods to save iptables rules, but some don't. Creating a script is guaranteed to work for anyone *(no guarantee of any kind is expressed or implied)*.

Add something like this to `/etc/rc.local` or a new script.

```
INTNET="eth1"
EXTNET="eth0"
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o $EXTNET -j MASQUERADE
iptables -A FORWARD -i $EXTNET -o $INTNET -j ACCEPT
iptables -A FORWARD -i $INTNET -o $EXTNET -j ACCEPT
```

and make sure it is executable (some distributions enable and disable processing of the file through the executable bit):

```
chmod a+x /etc/rc.local
```