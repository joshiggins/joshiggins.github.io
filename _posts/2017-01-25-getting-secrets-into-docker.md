---
layout: post
comments: true
title: "Getting secrets (and other assets) into Docker containers at runtime"
fulltitle: "Getting secrets (and other assets) into Docker containers at runtime"
excerpt: ""
categories : 
- docker
- containers
- secrets
- dnsmasq
---

We are making at push in order to containerise (if that's even a word) more parts of the HPC infrastructure for better manageability. Starting with fundamental aspects of any network connected system: name resolution and address assignment.

[dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) is a little server that provides both DNS and DHCP, optionally with network booting capability, *"for small networks"* (even though it can scale very well to large systems).

It makes sense for us to create a container for it, but should we also include all our context specific configuration in the container? The answer is unequivocally no. 

## Docker secrets

Docker has introduced support for [secrets](https://docs.docker.com/engine/reference/commandline/secret/) and while it is well suited for including passwords, API keys, and other information like that, the idea of including a binary blob to include more complex data than a string just feels messy.

It's also not clear how to persist these secrets, and if you want any kind of version control you have to do it yourself.

But the main problem is that it doesn't map well to the problem at hand. We need to supply contextual configuration for the service, that could include configuration files, network boot assets (kernels, etc) and maybe even a script that needs to run to fix things like permissions and all sorts.

## The solution? A fancy entrypoint script and a git repository

What we decided is that it would be good to store this configuration in some kind of repository, so why not make it a git repository?

When the container is starting, an entrypoint script pulls this repository and performs the required configuration, before handing over to the `dnsmasq` daemon. The only downside of this apporach is requiring the `git` package inside the container.

The repository, username and password should be passed in through environment variables or Docker secrets.

You can see an implementation here: [hpchud/dns](https://github.com/hpchud/dns). We put the private repository holding the configuration on bitbucket, and use an organisation API key to access it like so:

```
docker run -d --net=host \
    -e CONFIG_REPO=bitbucket.org/hpchud/dns-config.git \
    -e CONFIG_USER=<org name> \
    -e CONFIG_PASS=<api key> \
    hpchud/dns
```

The [`entrypoint.sh`](https://github.com/hpchud/dns/blob/master/entrypoint.sh) is straghtforward - after checking the required variables are available, we clone the repository and copy the config files into place (optionally running a `post.sh` script just before starting the daemon).

It works well. In addition, when restarting the container, the old config is cleaned out and pulled again - so that the information is updated on a `docker restart` without having to destroy and create a new container.

## Is this the Docker way?

I think that as long as

- we bomb out when the config repository is not available,
- the format of the config repository is well documented,

this is a good way to deal with the problem in line with the Docker philosophy.