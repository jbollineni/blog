---
title: Snapcast Server in a Container 
date: 2022-08-21
categories:
  - Multi-room Audio
tags:
  - Multi-room Audio
---

In this blog, let's look at using snapcast server software in a container. 

[Snapcast](https://github.com/badaix/snapcast) is a synchronous multiroom client-server audio player. While Snapcast server has multiple configurable options for stream sources, this setup will use a named `pipe` as a source, from which Snapcast reads PCM chunks of audio data. Using named `pipe` as a source extends the ability to configure multiple audio sources - like Librespot (see Librespot Docker container) to send audio output.

<!-- more -->

```
|Librespot|---write to---> /tmp/snapcast/fifo <---read from--- |snapcast server| ~~~ network ~~~ |[snapcast clients]|
```


## Setup

The snapcast-server docker image is available on [docker hub](https://hub.docker.com/repository/docker/jbollineni/snapcast-server). The container requires two inputs - a `snapserver.conf` file and `fifo` pipe file. The files on the filesystem are passed to the containers via docker volumes.

- Assuming `smarthome` is a your home-automation directory, create a directory with the name `snapcast` and create a file called `docker-compose.yml` with the following contents.


```
---
services:
  snapcast-server:
    image: jbollineni/snapcast-server:latest
    network_mode: "host"        
    restart: on-failure
    volumes:
      - /opt/smarthome/snapcast/snapserver.conf:/etc/snapserver.conf:ro
      - /tmp/snapcast/fifo:/tmp/snapcast/fifo
```

- Add another file called `snapserver.conf` to snapcast directory with the following contents, which tells snapcast server the path to the named `fifo` pipe as well as the audio sample format. See [Snapcast](https://github.com/badaix/snapcast) documentation for more options on setting up sources.


```
[stream]
stream = pipe:///tmp/snapcast/fifo?name=Librespot-docker&sampleformat=44100:16:2
```

*Note: The sample format is set to* `44100:16:2` *to match the* `Librespot` *source's sample format.* 

- Create a named pipe file


```
mkdir -p /tmp/snapcast
mkfifo /tmp/snapcast/fifo
```

## Run container

Navigate to the `snapcast` directory and run docker compose command

```
cd /opt/smarthome/snapcast
docker compose up -d
```

## Verify container

```
[violet@home snapcast]$ docker ps | grep snapcast
d8fd172f8f34   jbollineni/snapcast-server:latest   "/bin/sh -c 'rm /var…"   3 weeks ago   Up 3 weeks                       snapcast-snapcast-server-1
[violet@home snapcast]$
```



------

## Dockerfile

```
FROM alpine:edge

MAINTAINER Jana Bollineni (jana@neni.io)

LABEL version="1.2"
LABEL org.label-schema.name="Snapcast Server Docker" \
      org.label-schema.description="Snapcast server on alpine image with Avahi and D-Bus support" \
      org.label-schema.schema-version="1.0"

RUN apk -U add snapcast-server \
    && mkdir -p /tmp/snapcast/ \
    && apk add -U avahi \
    && apk add dbus \
    && dbus-uuidgen > /var/lib/dbus/machine-id \
    && mkdir -p /var/run/dbus \
    && rm -rf /etc/ssl /var/cache/apk/* /lib/apk/db/*

CMD dbus-daemon --config-file=/usr/share/dbus-1/system.conf --print-address; avahi-daemon -D; snapserver
```

