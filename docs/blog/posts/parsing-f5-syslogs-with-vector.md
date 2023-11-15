---
title: Parsing F5 syslogs with Vector
slug: parsing-f5-syslogs-with-vector
date: 2023-11-14
categories:
  - F5
  - Observability
tags:
  - F5, LTM
  - Vector
  - Observability, Syslog
draft: false
---

This article is part two of the series of articles and describes how to parse syslogs from F5 BIG-IPs.

While SNMP polling, Rest API calls, or F5 Telemetry iApp can be used to retrieve pool member status and state, these management calls are expensive and they add to the control plane CPU usage on the BIG-IP. BIG-IP LTM Syslogs events contain a wealth of details about the pool - like name, member IP:Port, member up/down health monitor status, member enabled/disabled administrative state, along with timestamps.

<!-- more -->

## Sample Pool logs

### SLB

```
*Pool Member Up*

<133>Nov 13 04:18:56 bigip01.example.net notice mcpd[7128]: 01070727:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 monitor status up. [ /Common/m_foo-stg_80_http: up ]  [ was down for 0hr:1min:37sec ]

*Pool Member Down*

<133>Nov 13 04:19:15 bigip01.example.net notice mcpd[7128]: 01070638:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 monitor status down. [ /Common/m_foo-stg_80_http: down; last error: /Common/m_foo-stg_80_http: Unable to connect @2023/11/13 04:19:15.  ]  [ was up for 0hr:0min:19sec ]

*Pool Member Disabled*

<133>Nov 13 04:01:51 bigip01.example.net notice mcpd[7128]: 01070639:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 session status forced disabled.

*Pool Member Enabled*

<133>Nov 13 04:06:16 bigip01.example.net notice mcpd[7128]: 01070639:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 session status enabled.

*Pool Member Forced Offline*

<133>Nov 13 04:17:19 bigip01.example.net notice mcpd[7128]: 01070638:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 monitor status forced down. [ /Common/m_foo-stg_80_http: up ]  [ was up for 1525hrs:55mins:10sec ]

```

### GSLB

```
*Pool Member Up*

<145>Nov 10 00:56:49.244 bigip01.example.net alert gtmd[22463]: 011a4002:1: SNMP_TRAP: Pool /Common/p_foo-stg-1t_80 member vs_10_1_1_51_80 (ip:port=10.1.1.51:80) state change red --> green

*Pool Member Down*

<145>Nov 10 01:01:49.615 bigip01.example.net alert gtmd[22463]: 011a4003:1: SNMP_TRAP: Pool /Common/p_foo-stg-1t_80 member vs_10_1_1_51_80 (ip:port=10.1.1.51:80) state change green --> red ( Monitor /Common/m_foo-stg-1t_80_http from 192.168.6.10 : state: connect failed)

*Pool Member Enabled*

<145>Nov 10 00:46:58.444 bigip01.example.net alert tmm1[27599]: 011a4005:1: SNMP_TRAP: Pool (/Common/p_foo-stg-1t_80) member (10.1.1.51:80) enabled

*Pool Member Disabled*

<145>Nov 10 00:40:25.313 bigip01.example.net alert tmm1[27599]: 011a4004:1: SNMP_TRAP: Pool (/Common/p_foo-stg-1t_80) member (10.1.1.51:80) disabled
```

## Vector Grok Parser
Vector uses [Vector Remap Language(VRL)](https://vector.dev/docs/reference/configuration/transforms/remap/) language which provides several [functions](https://vector.dev/docs/reference/vrl/functions/) and [expressions](https://vector.dev/docs/reference/vrl/expressions/) for transforming observability data. While Vector supports multiple parsing functions, this article will discuss the [parse_grok](https://vector.dev/docs/reference/vrl/functions/#parse_grok) and [parse_groks](https://vector.dev/docs/reference/vrl/functions/#parse_groks) functions. 

<!-- https://logz.io/blog/logstash-grok/-->

<!-- https://docs.mezmo.com/2.8/telemetry-pipelines/using-grok-to-parse -->

<!-- https://github.com/hpcugent/logstash-patterns/blob/master/files/grok-patterns
https://stackoverflow.com/questions/3075130/what-is-the-difference-between-and-regular-expressions
https://logz.io/blog/logstash-grok/ -->


Grok is used to parse unstructured data such as syslogs and derive structured data by combining text patterns. It uses a syntax `%{PATTERN:label}`, where `PATTERN` is a predefined or a named Grok pattern and `label` is a name of the field to store the matching captured data. The [logstash gork patterns](https://github.com/hpcugent/logstash-patterns/blob/master/files/grok-patterns) page is a good reference.


Following are some commonly used GROK patterns.

`GREEDYDATA` pattern translates to `.*` with which the regex engine tries to capture as many matches as possible (typically all characters in a string, except special characters such as \n or up until another expression is met.). 

`DATA` pattern translates to `.*?`. The regex engine tries to capture as few matches (non-greedy) as possible.


### Parsing an F5 Syslog Message

Consider the following syslog message generated when a health monitor marks a member as up. The goal is to capture the timestamp when the event occured, the hostname of the Big-IP that generated the event, pool name, member IP:Port, and the member's health status.

```bash
<133>Nov 13 04:18:56 bigip01.example.net notice mcpd[7128]: 01070727:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 monitor status up. [ /Common/m_foo-stg_80_http: up ]  [ was down for 0hr:1min:37sec ]

```

Run the command `vector vrl` to fire up the Vector VRL REPL (Read–eval–print loop).

- Set the syslog message as an event data

```
$ .data="<133>Nov 13 04:18:56 bigip01.example.net notice mcpd[7128]: 01070727:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 monitor status up. [ /Common/m_foo-stg_80_http: up ]  [ was down for 0hr:1min:37sec ]"
```
The above command will create a json object and can be verified by typing a `.` 

```
$ .
{ "data": "<133>Nov 13 04:18:56 bigip01.example.net notice mcpd[7128]: 01070727:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 monitor status up. [ /Common/m_foo-stg_80_http: up ]  [ was down for 0hr:1min:37sec ]" }
```


#### Iteration1

- Run parse_grok function

```
$ parse_grok!(.data, "%{GREEDYDATA:message}")
```

- Output
```
$ parse_grok!(.data, "%{GREEDYDATA:message}")
{ "message": "<133>Nov  7 01:30:07 bigip01.example.net notice mcpd[7128]: 01070727:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:443 monitor status up. [ /Common/m_foo-stg_80_http: up ]  [ was down for 0hr:1min:33sec ]" }
```

The `GREEDYDATA` pattern matches the entire string.


#### Iteration2

- Run parse_grok function

```
$ parse_grok!(.data, "%{GREEDYDATA:discard}%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{GREEDYDATA:message}")
```

- Output
```
{ "devicename": "bigip01.example.net", "discard": "<133>", "lbmonitortimestamp": "Nov 13 04:18:56", "message": "notice mcpd[7128]: 01070727:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 monitor status up. [ /Common/m_foo-stg_80_http: up ]  [ was down for 0hr:1min:37sec ]" }
```

The parse_grok function now extracted `<133>` into a field called `discard`, the timestamp in the log event to `lbmonitortimestamp` field, hostname to `deviename`` field and the remainder of the string into a field called `message`. Here `SYSLOGTIMESTAMP` and `HOSTNAME` are a predefined grok patterns.

#### Iteration3

- Run parse_grok function

```
$ parse_grok!(.data, "%{GREEDYDATA:discard}%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename}%{GREEDYDATA} Pool /Common/%{DATA:poolname} %{GREEDYDATA:message}")
```

- Output
```
{ "devicename": "bigip01.example.net", "discard": "<133>", "lbmonitortimestamp": "Nov 13 04:18:56", "message": "member /Common/10.1.1.51:80 monitor status up. [ /Common/m_foo-stg_80_http: up ]  [ was down for 0hr:1min:37sec ]", "poolname": "p_foo-stg_80" }
```

In this iteration, `GREEDYDATA` captures all characters up until it encounters `P` in Pool. Also, the `GREEDYDATA` is not followed by a label/identifier.

#### Iteration4

- Run parse_grok function

```
$ parse_grok!(.data, "%{GREEDYDATA}%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{GREEDYDATA} Pool /Common/%{GREEDYDATA:poolname} member /Common/%{GREEDYDATA:member} monitor status %{DATA:status}. %{GREEDYDATA}")
```

- Output
```
{ "devicename": "bigip01.example.net", "lbmonitortimestamp": "Nov 13 04:18:56", "member": "10.1.1.51:80", "poolname": "p_foo-stg_80", "status": "up" }
```

In this iteration, the monitor status is captured into the field `status`. The `DATA` pattern is used to be less greedy and capture until it encounters another expression marked by `GREEDYDATA`.

### Parsing with multiple grok patterns

The `parse_groks` function is used if log events of varying data need to be parsed. It uses an array of comma seperated patterns. The following log event shows a pool member admin state, different from the log event used above.

```
.data = "<133>Nov 13 04:06:16 bigip01.example.net notice mcpd[7128]: 01070639:5: Pool /Common/p_foo-stg_80 member /Common/10.1.1.51:80 session status enabled."
```

- Run parse_groks function
```
parse_groks!(
	.data,
    patterns: [
        "%{GREEDYDATA}%{SYSLOGTIMESTAMP:lbmonitortimestamp} %{HOSTNAME:devicename} %{GREEDYDATA} Pool /Common/%{GREEDYDATA:poolname} member /Common/%{GREEDYDATA:member} monitor status %{DATA:status}. %{GREEDYDATA}",
        "%{GREEDYDATA}%{SYSLOGTIMESTAMP:lbadmintimestamp} %{HOSTNAME:devicename} %{GREEDYDATA} Pool /Common/%{GREEDYDATA:poolname} member /Common/%{GREEDYDATA:member} session status %{GREEDYDATA:state}.",
		
        #Catch grok failed events
        "%{GREEDYDATA:grokfailedmessage}",
    ]
    )
```

- Output
```
{ "devicename": "bigip01.example.net", "lbmonitortimestamp": "Nov 13 04:06:16", "member": "10.1.1.51:80", "poolname": "p_foo-stg_80", "state": "enabled" }
```



## References

https://my.f5.com/manage/s/article/K16008

https://github.com/hpcugent/logstash-patterns/blob/master/files/grok-patterns

https://rbuckton.github.io/regexp-features/engines/oniguruma.html

https://stackoverflow.com/questions/26474873/how-do-i-match-a-newline-in-grok-logstash