---
title: Parsing F5 syslogs with Vector
slug: parsing-f5-syslogs-with-vector
date: 2023-10-01
categories:
  - F5
  - Observability
tags:
  - F5, LTM
  - Vector
  - Observability, Syslog
draft: true
---

This article is part two of the series of articles and describes how to parse syslogs from F5 BIG-IPs.

While SNMP polling, Rest API calls, or F5 Telemetry iApp can be used to retrieve pool member status and state, these management calls are expensive and they add to the control plane CPU usage on the BIG-IP. BIG-IP LTM Syslogs events contain a wealth of details about the pool - like name, member IP:Port, member up/down health monitor status, member enabled/disabled administrative state, along with timestamps.

<!-- more -->

## Sample Pool logs

### SLB

```
*Pool Member Up*

Oct 13 01:55:19 bigip01.example.net notice mcpd[5105]: 01070727:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status up. [ /Common/tcp: up ]  [ was down for 0hr:0min:44sec ]

*Pool Member Down*

Oct 13 01:55:40 bigip01.example.net notice mcpd[5105]: 01070638:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status down. [ /Common/tcp: down; last error: /Common/tcp: Unable to connect. @2023/10/13 01:55:40.  ]  [ was up for 0hr:0min:21sec ]

*Pool Member Disabled*

Oct 13 01:59:31 bigip01.example.net notice mcpd[5105]: 01070639:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 session status forced disabled.

*Pool Member Enabled*

Oct 13 02:19:42 bigip01.example.net notice mcpd[5105]: 01070639:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 session status enabled.

*Pool Member Forced Offline*

Oct 13 02:26:23 bigip01.example.net notice mcpd[5105]: 01070638:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status forced down. [ /Common/tcp: up ]  [ was up for 0hr:0min:12sec ]
```

### GSLB

## Vector Grok Parser
Vector uses [Vector Remap Language(VRL)](https://vector.dev/docs/reference/configuration/transforms/remap/) language which provides several [functions](https://vector.dev/docs/reference/vrl/functions/) and [expressions](https://vector.dev/docs/reference/vrl/expressions/) for transforming observability data. While Vector supports multiple parsing functions, this article will discuss the [parse_grok](https://vector.dev/docs/reference/vrl/functions/#parse_groks) function. 

<!-- https://logz.io/blog/logstash-grok/-->

<!-- https://docs.mezmo.com/2.8/telemetry-pipelines/using-grok-to-parse -->

<!-- https://github.com/hpcugent/logstash-patterns/blob/master/files/grok-patterns
https://stackoverflow.com/questions/3075130/what-is-the-difference-between-and-regular-expressions
https://logz.io/blog/logstash-grok/ -->


Grok is used to parse unstructured data such as syslogs and derive structured data by combining text patterns. It uses a syntax `%{PATTERN:label}`, where `PATTERN` is a predefined or a named Grok pattern and `label` is a name of the field to store the matching captured data. The [logstash gork patterns](https://github.com/hpcugent/logstash-patterns/blob/master/files/grok-patterns) page is a good reference.

### Parsing an F5 Syslog Message

Consider the following syslog message generated when a health monitor marks a member as up. The goal is to capture the timestamp when the event occured, the hostname of the Big-IP that generated the event, pool name, member IP:Port, and the member's health status.

```bash
Oct 13 01:55:19 bigip01.example.net notice mcpd[5105]: 01070727:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status up. [ /Common/tcp: up ]  [ was down for 0hr:0min:44sec ]
```

Run the command `vector vrl` to fire up the Vector VRL REPL (Read–eval–print loop).

- Set the syslog message as an event data

```
$ .data="Oct 13 01:55:19 bigip01.example.net notice mcpd[5105]: 01070727:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status up. [ /Common/tcp: up ]  [ was down for 0hr:0min:44sec ]"
```
The above command will create a json object and can be verified by typing a `.` 

```
$ .
{ "data": "Oct 13 01:55:19 bigip01.example.net notice mcpd[5105]: 01070727:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status up. [ /Common/tcp: up ]  [ was down for 0hr:0min:44sec ]" }
```


#### Iteration1

- Run parse_grok function

```
$ . = parse_grok!(.data, "%{GREEDYDATA:message}")
```

- Output
```
{ "message": "Oct 13 01:55:19 bigip01.example.net notice mcpd[5105]: 01070727:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status up. [ /Common/tcp: up ]  [ was down for 0hr:0min:44sec ]" }
```

The `GREEDYDATA` pattern translates to `.*` which matches the entire syslog string. The parse_grok function above matched the entire string and stored the string under a label called `message` at the root (represented by a .). `GREEDYDATA` is typically used to capture end or remainder of the string.


#### Iteration2

- Run parse_grok function

```
$ . = parse_grok!(.data, "%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{GREEDYDATA:message}")
```

- Output
```
{ "lbmonitortimestamp": "Oct 13 01:55:19", "message": "bigip01.example.net notice mcpd[5105]: 01070727:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status up. [ /Common/tcp: up ]  [ was down for 0hr:0min:44sec ]" }
```

The parse_grok function now extracted the timestamp in the log event into a field called `lbmonitortimestamp` and the remainder of the string into a field called `message`. `SYSLOGTIMESTAMP` is a predefined grok pattern.

#### Iteration3

- Run parse_grok function

```
$ . = parse_grok!(.data, "%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{LOGLEVEL:loglevel} %{GREEDYDATA:message}")
```

- Output
```
{ "devicename": "bigip01.example.net", "lbmonitortimestamp": "Oct 13 01:55:19", "loglevel": "notice", "message": "mcpd[5105]: 01070727:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status up. [ /Common/tcp: up ]  [ was down for 0hr:0min:44sec ]" }
```

In addition to timestamp, the parse_grok function bigip name to `devicename`, loglevel to `loglevel`, and the remainder of the string into a field called `message`. `SYSLOGTIMESTAMP`, `HOSTNAME`, `LOGLEVEL` are predefined grok patterns.

#### Iteration4

- Run parse_grok function

```
$ . = parse_grok!(.data, "%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{LOGLEVEL:loglevel} %{DATA:syslogprog}(:) %{DATA}(:) %{GREEDYDATA:message}")
```

- Output
```
{ "devicename": "bigip01.example.net", "lbmonitortimestamp": "Oct 13 01:55:19", "loglevel": "notice", "message": "Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 monitor status up. [ /Common/tcp: up ]  [ was down for 0hr:0min:44sec ]", "syslogprog": "mcpd[5105]" }
```

This iteration shows how the `DATA` pattern that translates to `.*?` captures the string until it encounters a `:` . (`:` is marked by `(:)` in regex). Notice that the first occurence of `DATA` pattern has a label, but the second occurennce does not. This is just way to ignore a field from being created.


#### Iteration5

- Run parse_grok function

```
$ . = parse_grok!(.data, "%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{DATA}(:) Pool /Common/%{DATA:poolname} member /Common/%{DATA:member} monitor status %{DATA:status}(.) %{GREEDYDATA}")
```

- Output
```
{ "devicename": "bigip01.example.net", "lbmonitortimestamp": "Oct 13 01:55:19", "member": "10.1.1.51:443", "poolname": "p_foo-stg_443", "status": "up" }
```

In this iteration, the 

#### Iteration6

This iteration uses log event with a admin enabled event

```
.data="Oct 13 02:19:42 bigip01.example.net notice mcpd[5105]: 01070639:5: Pool /Common/p_foo-stg_443 member /Common/10.1.1.51:443 session status enabled."
```

- Run parse_grok function

```
$ . = parse_grok!(.data, "%{SYSLOGTIMESTAMP:lbadmintimestamp} %{HOSTNAME:devicename} %{DATA}(:) Pool /Common/%{DATA:poolname} member /Common/%{DATA:member} session status %{DATA:state}(.)?$")
```

- Output

```
{ "devicename": "bigip01.example.net", "lbadmintimestamp": "Oct 13 02:19:42", "member": "10.1.1.51:443", "poolname": "p_foo-stg_443", "state": "enabled" }
```

