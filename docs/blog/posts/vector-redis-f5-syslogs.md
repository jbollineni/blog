---
title: Using Vector and Redis for F5 Syslogs
slug: vector-redis-f5-syslogs
date: 2023-11-13
categories:
  - Observability
  - F5 BIG-IP syslog
tags:
  - Vector
  - Redis
  - Observability, F5 BIG-IP Syslog

draft: false
---

This article is part one of a series discussing various components and configurations to parse pool-related syslogs from an F5 Big-IP and store the pool member state and status information in a Redis database. This article focuses on Vector and Redis.

<!-- more -->

[Part2: https://blog.neni.io/blog/parsing-f5-syslogs-with-vector/](https://blog.neni.io/blog/parsing-f5-syslogs-with-vector/)

[Part3: https://blog.neni.io/blog/display-f5-data-from-redis/](https://blog.neni.io/blog/display-f5-data-from-redis/)

## Elements

<br>

<div style="display: flex; justify-content: center;">
    <a href="vector-flow.png" class="image-popup"><img src="vector-flow.png" alt="vector-flow" title="vector-flow" width="600" height="500"></a>
</div>

### Vector

[Vector](https://vector.dev/docs/about/what-is-vector/) is an open-source observability data pipeline software that ingests, transforms, and routes logs and metrics. 

While Vector can be deployed in various roles in different topologies, this article focusses on deploying Vector in the role of an aggregator in a [Stream-based](https://vector.dev/docs/setup/deployment/topologies/#stream-based) topology. 

<br>
**Vector Components**

#### Sources

The [Source](https://vector.dev/docs/reference/configuration/sources/) component ingests logs from various sources like - Kafka, Stdin, logs sent via Syslog, etc. Following are examples of using a `UDP socket` or `Kafka` as a sources.

- UDP Socket Source
```toml
## SOURCE SECTION
[sources.my_source_id]
type = "socket"
address = "192.168.99.122:8014"
mode = "udp"
```

- Kafka Source
```toml
[sources.my_source_id]
bootstrap_servers = "broker1.example.com:9094,broker2.example.com:9094,broker3.example.com:9094"
group_id = "f5_logs"
topics = [ "f5.system.logs" ]
type = "kafka"

  [sources.my_source_id.tls]
  ca_file = "/path/to/certs/ca.crt"
  crt_file = "/path/to/certs/kafka-client.crt"
  key_file = "/path/to/certs/kafka-client.key"
  enabled = true
  verify_certificate = true
```

As a Kafka client, Vector connects to the Kafka broker endpoints and subscribes to a Kafka topic, to which various F5 devices would publish their logs. Typically, Kafka clients are provided with a TLS certificate, key and a CA cert for the client to perform mutual TLS auth with the Kafka broker endpoints.



#### Transforms

The [Transform](https://vector.dev/docs/reference/configuration/transforms/) component is where the data is transformed and shaped as needed. It takes a source component ID as an input. The event data is parsed, filtered, sampled, and enhanced using the `remap` type transform, which uses [VRL](https://vector.dev/docs/reference/configuration/transforms/remap/) language for processing the observability data.

```toml
# TRANSFORM SECTION
[transforms.my_transform_catch]
type = "remap"
inputs = [ "my_source_id" ]
source = '''
  #Drop any log events that do NOT contain the keyword 'Pool'
  if !match_any(string!(.message), [r'Pool'])
  {
  abort
  }

    #Replace occurences of "\n" character in the log event
    .message = replace(string!(.message), "\n", "")
  
	#Parsing log events that contain the keyword 'Pool'
  . = parse_groks!(
      string!(.message),
      patterns: [
        #SLB related parser
        "%{GREEDYDATA}%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{GREEDYDATA} Pool /Common/%{GREEDYDATA:poolname} member /Common/%{GREEDYDATA:member} monitor status %{DATA:status}. %{GREEDYDATA}",
        "%{GREEDYDATA}%{SYSLOGTIMESTAMP:lbadmintimestamp} %{HOSTNAME:devicename} %{GREEDYDATA} Pool /Common/%{GREEDYDATA:poolname} member /Common/%{GREEDYDATA:member} session status %{GREEDYDATA:state}.",

        #GSLB related parser
        "%{GREEDYDATA}%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{GREEDYDATA} Pool /Common/%{DATA:poolname} member %{DATA} \\(ip:port=%{DATA:member}\\) state change %{DATA:OLDstatus} --> %{DATA:status} %{GREEDYDATA}",
        "%{GREEDYDATA}%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{GREEDYDATA} Pool /Common/%{DATA:poolname} member %{DATA} \\(ip:port=%{DATA:member}\\) state change %{DATA:OLDstatus} --> %{DATA:status}\\\\%{GREEDYDATA}",

        #Catch all other events
        "%{GREEDYDATA:GROKFAILEDMESSAGE}",
    ]
  )
 
  del(.GROKFAILEDMESSAGE)
'''

[transforms.my_transform_reduced]
type = "remap"
inputs = [ "my_transform_catch" ]
source = '''
  #Drop empty events
  if is_empty(.) {
  	abort
  }
  #Drop events that have 'slot' as device name (Logs from vcmp guest devices)
  if contains(string!(.devicename), "slot") {
    abort
  }
'''

[transforms.my_transform_normalize]
type = "remap"
inputs = [ "my_transform_reduced" ]
source = '''
  .YEAR = join!(slice!(split(to_string(now()), "-",), start:0, end:1))

  if (exists(.lbmonitortimestamp)) {
      .lbmonitortimestamp_new_date_string, err = .YEAR + " " + .lbmonitortimestamp
      .lbmonitortimestamp, err = parse_timestamp(.lbmonitortimestamp_new_date_string, format: "%Y %b %d %X")
  }
  if (exists(.lbadmintimestamp)) {
      .lbadmintimestamp_new_date_string, err = .YEAR + " " + .lbadmintimestamp
      .lbadmintimestamp, err = parse_timestamp(.lbadmintimestamp_new_date_string, format: "%Y %b %d %X")
  }

  del(.YEAR)
  del(.lbmonitortimestamp_new_date_string)
  del(.lbadmintimestamp_new_date_string)

  if (.status) == "up" {
      .status = "Available"
  } else if  (.status) == "down" {
      .status = "Offline"
  } else if  (.status) == "red" {
      .status = "Offline"
  } else if (.status) == "green" {
      .status = "Available"
  } else if (.status) == "force" {
      .status = "Offline"
  } else if (.state) == "forced disabled" {
      .state = "Disabled"
  } else if (.state) == "enabled" {
      .state = "Enabled"
  }
'''
```

!!! note ""
    See [Part2: Parsing F5 syslogs with Vector](/blog/parsing-f5-syslogs-with-vector/) to understand Grok parsing in detail


#### Sinks

The [Sink](https://vector.dev/docs/reference/configuration/sinks/) component delivers the data to the destinations - like Elastic, Redis or just the console. It takes a transform component id as input.

In the following example, Vector uses a `redis` sink and publishes the observability data to a redis channel.

- Redis Sink
```toml
## SINK SECTION
[sinks.redis]
type = "redis"
inputs = [ "my_transform_normalize" ]
data_type = "channel"
endpoint = "redis://localhost:6379/0"

key = "vector"

  [sinks.redis.encoding]
  codec = "json"
```

- Console Sink
```toml
#[sinks.console]
#  inputs = ["my_transform_normalize"]
#  type = "console"
#  target = "stdout"
#  encoding.codec = "json"
```

#### Running Vector

Add the source, transforms, and sink sections to a config file named `vector.toml` and run Vector pointing to the config file. For other options refer to [documentation](https://vector.dev/docs/reference/configuration/#formats).

```bash
/usr/bin/vector -c /etc/vector/config.d/vector.toml
```

If Vector is installed using a package manager, a systemd service unit config file `/etc/systemd/system/vector.service` should be installed too. Update the service unit config file with the path to `vector.toml` config file, reload the systemd daemon and start the service.

*Example:*


```bash
[red@vector-srv01 ~]$ systemctl status vector
● vector.service - Vector
   Loaded: loaded (/etc/systemd/system/vector.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-10-03 02:44:58 UTC; 9s ago
     Docs: http://vector.dev
 Main PID: 16787 (vector)
    Tasks: 18
   Memory: 21.6M
   CGroup: /system.slice/vector.service
           └─16787 /usr/bin/vector -c /etc/vector/config.d/vector.toml

Oct 03 02:44:58 vector-srv01.example.com systemd[1]: Started Vector.
Oct 03 02:44:58 vector-srv01.example.com vector[16787]: 2023-10-03T02:44:58.436207Z  INFO vector::app: Log level is enabled. le...info"
Oct 03 02:44:58 vector-srv01.example.com vector[16787]: 2023-10-03T02:44:58.437999Z  INFO vector::app: Loading configs. paths=[...oml"]
Oct 03 02:44:58 vector-srv01.example.com vector[16787]: 2023-10-03T02:44:58.480779Z  INFO vector::topology::running: Running he...ecks.
```


### Redis
Redis is an in-memory data structure store. It can be used as both a message broker - to publish and subscribe to events, and/or a database - to store structured data.

Vector's Sink section shown above uses Redis sink to publish the events to the channel datatype of the Redis instance. 

A Redis client is required to subscribe to the Redis channel, receive messages, and then write data to a Redis database.

#### Redis Client script

```py
import redis
import json

def redis_client():
    #redis connection object
    r = redis.Redis(host="localhost", port=6379, db=1)
    p = r.pubsub()
    #subscribe to 'vector' channel
    p.subscribe('vector')

    for message in p.listen():
        if message['type'] == 'message':
            data = json.loads(message['data'].decode('utf-8'))
            if "lbmonitortimestamp" in data:
                r.hset(
                    f"{data['devicename']}:{data['poolname']}", 
                    f"{data['member']}~status", data['status'],
                    f"{data['member']}~monitortimestamp", data['lbmonitortimestamp'] 
                )
            elif "lbadmintimestamp" in data:
                r.hset(
                    f"{data['devicename']}:{data['poolname']}", 
                    f"{data['member']}~state", data['state'],
                    f"{data['member']}~admintimestamp", data['lbadmintimestamp'] 
                )

if __name__ == "__main__":
    redis_client()
```

The Redis client above subscribes to Redis channel named `vector`, reads the data and stores it in the database. A [hash](https://redis.io/docs/data-types/hashes/) data structure stores data in a field-value format for a given key. Hashes provide the ability to add/modify/delete field pairs without the need to retrieve the existing data.

Following is an example of hash datastore when viewed using [RedisInsight](https://redis.com/redis-enterprise/redis-insight/).

<br>

<div style="display: flex; justify-content: center;">
    <a href="redisinsight.png" class="image-popup"><img src="redisinsight.png" alt="redisinsight" title="redisinsight" width="800" height="500"></a>
</div>


[Part2: Parsing F5 syslogs with Vector](/blog/parsing-f5-syslogs-with-vector/)