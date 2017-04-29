---
layout: post
comments: true
title: "Randomise the Docker bridge network address"
fulltitle: ""
excerpt: ""
categories : 
- hpc
- docker
- containers
- notes
- howto
---

If you are running Docker on a large number of hosts, the daemon will probably assign itself the same bridge network address on many of them.

This can cause some problems for any software that will aggressively discover and utilise any connected network interface for communication - such as MPI.

In this case, it looks at the Docker bridge and assumes that it can use it for inter-host communication, which is not the case.

To solve this, you can configure each Docker host to have a different bridge network using the `--bip=` daemon flag.

We have Docker running on around 80 hosts and I did not fancy managing the bridge networks by hand. They boot diskless and all address assignment is automatic. So I added the following snippit to the init script

```
DOCKERNET=10.$(( $RANDOM % 253 )).$(( $RANDOM % 253 )).1
exec /usr/bin/dockerd --bip=${DOCKERNET}/24
```

This will generate a _random enough_ /24 subnet and assign the Docker bridge the first host address, e.g. `10.28.145.1`.

Your mileage may vary but this strategy works for me. I did consider generating a bridge network address based on the automatically assigned IP for the host, but this turn out quite complicated and I wanted to keep it a plain `sh` script.