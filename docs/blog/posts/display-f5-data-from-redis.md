---
title: Display F5 Data from Redis
slug: display-f5-data-from-redis
date: 2023-12-17
categories:
  - F5 BIG-IP syslog
  - FastAPI
  - Python
  - Redis
tags:
  - Redis
  - F5 BIG-IP Syslog
  - FastAPI
  - Python

draft: false
---

This article is part three of a series, and it describes steps to retrieve F5 BigIP pool member information from a Redis database and return the data either in a svg image format or a json format, over an HTTP API call. The svg image format can be used to embed the image in an HTML page or a Github README.md page or the json endpoint can be called via a javascript.

<!-- more -->

[Part1: https://blog.neni.io/blog/vector-redis-f5-syslogs/](https://blog.neni.io/blog/vector-redis-f5-syslogs/)

[Part2: https://blog.neni.io/blog/parsing-f5-syslogs-with-vector/](https://blog.neni.io/blog/parsing-f5-syslogs-with-vector/)

## Components

- Python App for HTTP API based on [FastAPI](https://fastapi.tiangolo.com/)

- [Redis](vector-redis-f5-syslogs#redis): Database where the `wip`, `vip` or `pool` data is stored.

## Designing an API endpoint

The status API endpoint should

- provide data using a HTTP GET method over an API

- support output in `json` and `svg` image formats.

- Support querying based on parameters like `wip`, `vip` or `pool`.


## Set up Environment

## Clone the github repository

The app code is available at [https://github.com/jbollineni/f5-poolmember-dashboard](https://github.com/jbollineni/f5-poolmember-dashboard). Clone the repository to get started.

```bash
git clone git@github.com:jbollineni/f5-poolmember-dashboard.git
cd f5-poolmember-dashboard
```

### Python virtual environment
Create and activate python virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install python modules

```bash
pip install -r requirements.txt
```

### Temporary server
Start a temporary ASGI server with uvicorn

```{.bash }
./start_uvicorn_server.sh 
```


### View Data in Redis

#### Pool database (db1)

The [redis-client](https://blog.neni.io/blog/vector-redis-f5-syslogs/#redis-client-script) script populates Redis db1 with real-time pool member data that is parsed from F5 syslogs.

<div style="display: flex; justify-content: center;">
    <a href="redis-db1.png" class="image-popup"><img src="redis-db1.png" alt="redisinsight" title="redisinsight" width="800" height="500"></a>
</div>


#### VIP database (db3)

Redis db3 is populated with SLB VIP name to pool mappings and is used to look up the pool name using the vip name. Similarly, db2 is used to pupulate GSLB WIP to pool mappings.

<div style="display: flex; justify-content: center;">
    <a href="redis-db3.png" class="image-popup"><img src="redis-db3.png" alt="redisinsight" title="redisinsight" width="800" height="500"></a>
</div>



### Get Data over an API

Navigate to the URL shown in the output of uvicorn to access the site. Access the swagger page via `/docs` or OpenAPI page `/redoc` for more information on API parameters.

#### Table format

<div style="display: flex; justify-content: center;">
    <a href="slbvip_status_table.png" class="image-popup"><img src="slbvip_status_table.png" alt="redisinsight" title="redisinsight" width="800" height="500"></a>
</div>

#### json format

<div style="display: flex; justify-content: center;">
    <a href="slbvip_status_json.png" class="image-popup"><img src="slbvip_status_json.png" alt="redisinsight" title="redisinsight" width="800" height="500"></a>
</div>


### Running in production

To run the app in production, either setup the app as a systemd service or build a docker container image and run it as a container. Add an Nginx proxy as a front-end to handle TLS offload and certificate management, and to also expose port 443 to clients.
