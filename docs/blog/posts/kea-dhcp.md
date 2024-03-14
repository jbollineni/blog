---
title: KEA DHCP as a container
slug: kea-dhcp-as-a-container
date: 2024-01-30
categories:
  - KEA
  - DHCP
  - Docker
tags:
  - KEA
  - DHCP
  - Docker
draft: false
---
KEA DHCP is a newer, modular and extensible DHCP software from ISC. One of its core functionality is online reconfiguration over REST APIs, without the need to stop and start the DHCP service. See https://www.isc.org/kea/ for details. This article describes steps to standup a simple KEA DHCP server as a container for testing or serving leases for a small environment.


<!-- more -->

## Building a docker image

### Dockerfile

```Dockerfile
FROM ubuntu:22.04

RUN set -eux;  \
  apt-get update \
  &&apt-get install -y bash vim curl net-tools apt-transport-https  \
  && curl -1sLf 'https://dl.cloudsmith.io/public/isc/kea-2-3/setup.deb.sh'   |  distro=ubuntu codename=jammy arch=some-arch bash \
  && apt-get update \
  && apt-get install -y isc-kea \
  && apt-get remove -y curl apt-transport-https \
  && apt-get -q -y clean \
  && rm -rf /var/lib/apt/lists/* \
  && touch /var/lib/kea/dhcp4.leases

CMD /usr/sbin/kea-dhcp4 -d -c /etc/kea/kea-dhcp4.conf
```

*Note: If the docker image is being built on a host in an environment with no direct Internet access, try using a proxy. Add the following ENV instructions with the proxy endpoints in the Dockerfile, before the RUN instruction.*

```
ENV http_proxy http://proxy-endpoint.example.net:3128
ENV https_proxy http://proxy-endpoint.example.net:3128
```



### Build
Build a docker image using the above Dockerfile

```bash
red@services:~/docker-build$ docker build -t jbollineni/kea-dhcp:v2.3.3 .

```

View the docker images.

```bash
red@services:~/docker-build$ docker image ls
REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
jbollineni/kea-dhcp                 v2.3.3    3dfb153b36ae   7 seconds ago   175MB
```

## Simplified Config File

KEA DHCP uses json structure for configuration, providing extensibility to fetch/update config over REST APIs. Following is an example config to serve a least for a specific mac address. Refer to the [documentation](https://kea.readthedocs.io/en/latest/arm/config.html) for information.

```json
{

# DHCPv4 configuration starts on the next line
"Dhcp4": {

# First we set up global values
  "valid-lifetime": 4000,
  "renew-timer": 1000,
  "rebind-timer": 2000,

# Next we set up the interfaces to be used by the server.
  "interfaces-config": {
      "interfaces": [ "eth0" ]
  },

# And we specify the type of lease database
  "lease-database": {
      "type": "memfile",
      "persist": true,
      "lfc-interval": 3600,
      "name": "/var/lib/kea/dhcp4.leases"
  },

# Finally, we list the subnets from which we will be leasing addresses.
  "subnet4": [ {
          "subnet": "10.10.1.0/25",
          "option-data":[
              {
                  "name": "routers",
                  "data": "10.10.1.1"
              },
              {
                  "name": "domain-name-servers",
                  "data": "192.168.8.8"
              }
              ],

          "reservations": [
              {
                  "hw-address": "b8:83:03:7b:f3:14",
                  "ip-address": "10.10.1.20",
                  "hostname": "sw01",
                  "option-data": [
                      {
                         "name": "v4-captive-portal",
                         "data": "http://reposerver.example.net/sonic/images/sonic.bin"
                      }
                  ]
              }
              ]
      }
  ]

  }
}
```

## Directory Layout

Following is the directory structure of docker compose and KEA DHCP files.

```bash
red@services:~/kea-dhcp$ tree
.
├── config
│   ├── kea-dhcp4.conf
├── data
│   └── dhcp4.leases
└── docker-compose.yml

```

## Docker compose file

The volumes defined in docker compose file map the load 

```bash
red@services:~/kea-dhcp$ cat docker-compose.yml 
---
version: "3"
services:
  kea-dhcp:
    container_name: kea-dhcp
    image: jbollineni/kea-dhcp:v2.3.3
    network_mode: "host"
    restart: unless-stopped
    volumes:
      - ./config/kea-dhcp4.conf:/etc/kea/kea-dhcp4.conf
      - ./data/dhcp4.leases:/var/lib/kea/dhcp4.leases

```

## Running the container

Bring up the container using the `docker compose up` command. This should show the container starup logs as well other information such as DHCP discovery requests, DHCP lease allocations, etc.

```bash
red@services:~/kea-dhcp$ docker compose up
```

*Note: Use `docker compose up -d` to run and detach from the container, and `docker compose down` to stop the container.*


## View running Containers & Processes

```bash
red@services:~/kea-dhcp$ docker ps
CONTAINER ID   IMAGE                                      COMMAND                  CREATED         STATUS         PORTS     NAMES
dd4ec0741db4   jbollineni/kea-dhcp:v2.3.3   "/bin/sh -c '/usr/sb…"   3 minutes ago   Up 2 minutes             kea-dhcp
```

```bash
red@services:~/kea-dhcp$ sudo netstat -anup
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name  
udp        0      0 10.10.1.25:67         0.0.0.0:*                           20924/kea-dhcp4 
udp        0      0 0.0.0.0:68              0.0.0.0:*                           20840/dhclient  
udp        0      0 0.0.0.0:68              0.0.0.0:*                           20839/dhclient  
```

```bash
red@services:~/kea-dhcp$ ps faux     
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root     20889  0.0  0.1 1452884 11632 ?       Sl   03:08   0:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id a5849578bf9815ece57449ed04704ce19a6a56ee20df374de95782a7b9d9813a -ad
root     20910  0.0  0.0   2884   972 ?        Ss   03:08   0:00  \_ /bin/sh -c /usr/sbin/kea-dhcp4 -d -c /etc/kea/kea-dhcp4.conf
root     20924  0.0  0.2  71440 17040 ?        Sl   03:08   0:00      \_ /usr/sbin/kea-dhcp4 -d -c /etc/kea/kea-dhcp4.conf
    
```


## References

https://www.isc.org/kea/

https://kea.readthedocs.io/en/latest/arm/config.html
