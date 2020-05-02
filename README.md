# Docker Homebridge Zwave

This Alpine/Ubuntu Linux based Docker image allows you to run [Nfarina's](https://github.com/nfarina) [Homebridge](https://github.com/nfarina/homebridge) on your home network which emulates the iOS HomeKit API. It is a fork of the main [docker-homebridge](https://github.com/oznu/docker-homebridge) repo with added support for native ZWave devices via [open-zwave](https://github.com/OpenZWave/open-zwave) and [homebridge-openzwave](https://www.npmjs.com/package/homebridge-openzwave)

## Why a Fork?
`docker-homebridge` recommends adding in additional packages during the image `startup.sh` script. Getting OpenZWave working requires building from source (and is fairly tricky on Alpine). Packaging this as a Docker image allows for much faster boots/restarts of the Docker container and makes it a bit easier to get started.

This fork is primarily for my own usage at home, where I run homebridge in my homelab. I aim to keep it up to date with the main [docker-homebridge](https://github.com/oznu/docker-homebridge) as much as possible (but PRs always welcome if I miss a version)!

As of now, I have removed support for [raspbian](https://github.com/oznu/docker-homebridge/blob/master/raspbian-installer.sh) and [Dockerfile.ubuntu](https://github.com/oznu/docker-homebridge/blob/master/Dockerfile.ubuntu). _Note: The image will run just fine on Ubuntu, so don't let that stop you! I'm just not supporting people who want the Docker image itself to run Ubuntu rather than Alpine._

I've also removed support for the [https://github.com/oznu/docker-homebridge#5-logs-showing-service-name-conflict-or-host-name-conflict](no-avahi) version of the image. This is primarily to reduce complexity for my own work - I'd welcome a PR adding support for this back in, and I may get around to it some point in the future :)

## Resources
_these all link to the primary repository, there's nothing special here_

  * [Guides](#guides)
  * [Compatibility](#compatibility)
  * [Usage](#usage)
  * [Parameters](#parameters)
  * [Homebridge Config](#homebridge-config)
  * [Installing Plugins](#homebridge-plugins)
  * [Docker Compose](#docker-compose)
  * [Troubleshooting](#troubleshooting)

## Guides

- [Running Homebridge on a Synology NAS](https://github.com/oznu/docker-homebridge/wiki/Homebridge-on-Synology)

## Compatibility

Homebridge requires full access to your local network to function correctly which can be achieved using the ```--net=host``` flag.
Currently this image will not work when using [Docker for Mac](https://docs.docker.com/docker-for-mac/) or [Docker for Windows](https://docs.docker.com/docker-for-windows/) due to [this](https://github.com/docker/for-mac/issues/68) and [this](https://github.com/docker/for-win/issues/543).


## Usage

```shell
docker run \
  --net=host \
  --name=homebridge \
  -e PUID=<UID> -e PGID=<GID> \
  -e TZ=<timezone> \
  -e HOMEBRIDGE_CONFIG_UI=1 \
  -e HOMEBRIDGE_CONFIG_UI_PORT=8080 \
  -v </path/to/config>:/homebridge \
  seanherron/homebridge-zwave
```

## Parameters

The parameters are split into two halves, separated by a colon, the left hand side representing the host and the right the container side.

* `--net=host` - Shares host networking with container, **required**
* `-v /homebridge` - The Homebridge config and plugin location
* `-e TZ` - for [timezone information](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) e.g. `-e TZ=Europe/London`
* `-e PGID` - for GroupID - see below for explanation
* `-e PUID` - for UserID - see below for explanation

##### *Optional Settings:*

* `-e PACKAGES` - Additional [packages](https://pkgs.alpinelinux.org/packages) to install (comma separated, no spaces) e.g. `-e PACKAGES=openssh`
* `-e TERMINATE_ON_ERROR=1` - If `TERMINATE_ON_ERROR` is set to `1` then the container will exit when the Homebridge process ends, otherwise it will be restarted.
* `-e HOMEBRIDGE_INSECURE=1` - Start homebridge in insecure mode using the `-I` flag.
* `-e HOMEBRIDGE_DEBUG=1` - Enable debug level logging using the `-D` flag.

##### *Homebridge UI Options*:

This is the only supported method of running [homebridge-config-ui-x](https://github.com/seanherron/homebridge-zwave-config-ui-x) on seanherron/homebridge-zwave.

* `-e HOMEBRIDGE_CONFIG_UI=1` - Enable and configure [homebridge-config-ui-x](https://github.com/seanherron/homebridge-zwave-config-ui-x) which allows you to manage and configure Homebridge from your web browser.
* `-e HOMEBRIDGE_CONFIG_UI_PORT=8080` - The port to run [homebridge-config-ui-x](https://github.com/seanherron/homebridge-zwave-config-ui-x) on. Defaults to port 8080.

### User / Group Identifiers

Sometimes when using data volumes (`-v` flags) permissions issues can arise between the host OS and the container. We avoid this issue by allowing you to specify the user `PUID` and group `PGID`. Ensure the data volume directory on the host is owned by the same user you specify and it will "just work".

In this instance `PUID=1001` and `PGID=1001`. To find yours use `id user` as below:

```
  $ id <dockeruser>
    uid=1001(dockeruser) gid=1001(dockergroup) groups=1001(dockergroup)
```

## Homebridge Config

The Homebridge config file is located at ```</path/to/config>/config.json```
This file will be created the first time you run the container if it does not already exist.

## Homebridge Plugins

Plugins should be defined in the ```</path/to/config>/package.json``` file in the standard NPM format.
This file will be created the first time you run the container if it does not already exist.

Any plugins added to the `package.json` will be installed each time the container is restarted.
Plugins can be uninstalled by removing the entry from the `package.json` and restarting the container.

You can also install plugins using `npm` which will automatically update the package.json file as you add and remove modules.

**You must restart the container after installing or removing plugins for the changes to take effect.**

### To add plugins using npm:

```
docker exec <container name or id> npm install <module name>
```

Example:

```
docker exec homebridge npm install homebridge-dummy
```

### To remove plugins using npm:

```
docker exec <container name or id> npm remove <module name>
```

Example:

```
docker exec homebridge npm remove homebridge-dummy
```

### To add plugins using `startup.sh` script:

The first time you run the container a script named [`startup.sh`](/root/defaults/startup.sh) will be created in your mounted `/homebridge` volume. This script is executed everytime the container is started, before Homebridge loads, and can be used to install plugins if you don't want to edit the `package.json` file manually.

To add plugins using the `startup.sh` script just use the `npm install` command:

```shell
#!/bin/sh

npm install homebridge-dummy
```

This container does **NOT** require you to install plugins globally (using `npm install -g` or `yarn global add`) and doing so is **NOT** recommended or supported.

## Docker Compose

If you prefer to use [Docker Compose](https://docs.docker.com/compose/):

```yml
version: '2'
services:
  homebridge:
    image: seanherron/homebridge-zwave:latest
    restart: always
    network_mode: host
    environment:
      - TZ=Australia/Sydney
      - PGID=1000
      - PUID=1000
      - HOMEBRIDGE_CONFIG_UI=1
      - HOMEBRIDGE_CONFIG_UI_PORT=8080
    volumes:
      - ./volumes/homebridge:/homebridge
```

## Troubleshooting

#### 1. Verify your config.json and package.json syntax

Many issues appear because of invalid JSON. A good way to verify your config is to use the [jsonlint.com](http://jsonlint.com/) validator.

#### 2. When running on Synology DSM set the `DSM_HOSTNAME` environment variable

You may need to provide the server name of your Synology NAS using the `DSM_HOSTNAME` environment variable to prevent [hostname conflict errors](https://github.com/oznu/docker-homebridge/issues/35). The value of the `DSM_HOSTNAME` environment should exactly match the server name as shown under `Synology DSM Control Panel` -> `Info Centre` -> `Server name`, it should contain no spaces or special characters.

#### 3. Need ffmpeg?

ffmpeg, with `libfdk-aac` audio support is included in this image.

```

See the wiki for a list of image variants: https://github.com/oznu/docker-homebridge/wiki

#### 6. Ask on Discord

Join the [Official Homebridge Discord](https://discord.gg/Cmq8a44) community and ask in the [#docker](https://discord.gg/Cmq8a44) channel.

## License

docker-homebridge-zwave is a fork of docker-homebridge. (https://github.com/oznu/docker-homebridge) by oznu, copyright 2017-2020. Remaining pieces are by Sean Herron, copyright 2020.

This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the [GNU General Public License](./LICENSE) for more details.
