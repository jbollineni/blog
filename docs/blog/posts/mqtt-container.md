---
title: Containerizing Home Automation
date: 2023-05-04
categories:
  - links
tags:
  - links
published: False
draft: true
---

<!-- more -->
MQTT Docker:

Eclipse Mosquitto provides an official docker image on Docker [HUB](https://hub.docker.com/_/eclipse-mosquitto)

~~~ bash
[violet@greenbox ~]$ docker ps -f name=elated_lumiere  -s
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES               SIZE
164cf486ee32        eclipse-mosquitto   "/docker-entrypoint.…"   4 days ago          Up 4 days                               elated_lumiere      0B (virtual 5.6MB)
[violet@greenbox ~]$ 
~~~

~~~ bash
docker run -d \
  --net=host \
  --name=mosquitto \
  -v source="/opt/mosquitto/config",destination=/mosquitto/config/ \
  -v source="/opt/mosquitto/data",destination=/mosquitto/data/ \
  -v source="/var/log/mosquitto/",destination=/mosquitto/log/ \
  eclipse-mosquitto:latest
~~~

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0NjIwMjIyMzhdfQ==
-->