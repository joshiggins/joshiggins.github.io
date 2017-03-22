---
layout: post
comments: true
title: "The easiest way to install, setup and run Nagios and Nagvis on Linux"
fulltitle: "The easiest way to install, setup and run Nagios and Nagvis on Linux"
excerpt: ""
categories : 
- docker
- containers
- nagios
- howto
---

This article will show you the easiest method to install, setup and run Nagios and Nagvis for production using Docker.

None of the configuration will be stored inside the container image itself - it will be cloned from a (private) git repository when the container is launched. This way, you don't have to maintain and recreate images when you want to change a configuration. For info on how this is implemented inside the image, see this [post](/2017/getting-secrets-into-docker/).

## Step 0

Install [Docker](https://docs.docker.com/engine/installation/).

## Step 1

Create a git repository that will hold the configuration for Nagios, and your Nagvis maps.

Clone the [joshiggins/nagios-config-template](https://github.com/joshiggins/nagios-config-template) repository and delete all the git history. Then, initialise the folder again as a new git repository and push to your service of choice. In this example, I am using Bitbucket.

```
$ git clone https://github.com/joshiggins/nagios-config-template.git
$ cd nagios-config-template.git
$ rm -rf .git
$ git init .
$ git add *
$ git commit -a -m 'first commit'
$ git remote add origin https://joshiggins@bitbucket.org/hpchud/nagios-config.git
$ git push -u origin master
```

Inside this repository you will place your configuration. Configuring Nagios is well covered online, just have a look. The [documentation](https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/4/en/config.html) might be the best place to start.

The Nagios `etc` folder is stored in the repository under `./nagios`.

The Nagvis `maps` folder is stored under `./nagvis/maps`. The Nagvis `userdata` folder is stored under `./nagvis/userdata`.

## Step 2

Run the [hpchud/nagios](https://hub.docker.com/r/hpchud/nagios/) image using your configuration repository. If you need to specify a username and password (or organisation name and API key) you can do so using environment variables.

```
docker run -d -p 80:80 \
    -e CONFIG_REPO=bitbucket.org/hpchud/nagios-config.git \
    -e CONFIG_USER=hpchud
    -e CONFIG_PASS=<api key> \
    hpchud/nagios
```

The user interfaces will now be available at

[localhost/nagios](http://localhost/nagios) and
[localhost/nagvis](http://localhost/nagvis) respectively.

That's it. If you have any problems, drop me a line or [create an issue](https://github.com/hpchud/nagios/issues) on GitHub.

## Persisting changes at runtime

It can be difficult to write map definitions for Nagvis by hand. You will probably want to use the web interface to set up the maps once your Nagios is configured properly. But if you do this, the changes will be lost when the container is restarted and the configuration is loaded from the repository again.

Therefore, we need to save the changes made at runtime back to the configuration repository.

To do this, you can simply enter the container's environment using the command

```
docker exec -it cid /bin/bash
```

where `cid` is the ID or name of the container. Then you can navigate to `/tmp/nagios-config` and simply `git commit` and `git push` the changes.

### Alternative method to persist changes at runtime

First clone the configuration repository to your machine. We will call this `$CONFIG_PATH`.

Instead of launching the container and specifying a repository to download, we will start a container with no configuration and mount the folders from our local clone of the repository into the container as [Docker volumes](https://docs.docker.com/engine/tutorials/dockervolumes/). This means that changes made at runtime will be made directly into your local clone of the configuration repository.

This is really useful in production, because you can make and test configuration changes by simply spinning up another instance of the container before commiting the changes to the repository and restarting the main instance.

To start a container with the configuration mounted through a Docker volume:

```
docker run -d -p 80:80 \
    -v $CONFIG_PATH/nagios:/usr/local/nagios/etc \
    -v $CONFIG_PATH/nagvis/maps:/usr/local/nagivs/etc/maps \
    -v $CONFIG_PATH/nagvis/userfiles:/usr/local/nagvis/share/userfiles \
    hpchud/nagios
```

After setting up the maps, check back in your `$CONFIG_PATH` where you made a local clone of the configuration repository. It should now contain the changes you just made, and you can push them. This container can be thrown away once you've done changing things.