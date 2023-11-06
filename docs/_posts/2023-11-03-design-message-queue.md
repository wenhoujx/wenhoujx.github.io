---
layout: post
title:  "设计一个distributed message queue"
date:   2023-10-27 12:28:00 -0400
categories: system-design message-queue kafka
published: false
---

**问题**:

system design 如何设计一个 distributed `message queue`

## 需求

既然是 distributed 就一定要问一下的:

- cap
  - 牺牲 availability, 意味着有时service will be down
  - 牺牲 consistency, 意味着有时候 data will be lost

既然是message queue, 我们要问一下 throughput:

- 有多少message
- 每小时有多少message
- 有多少consumer

## highlevel

message queue基本有以下几个部分:

- producer client
- broker
- zookeeper for leader election and failover
- consumer client

### producer -> broker -> consumer

producer 把 message 发给 broker, API 如下:

```sh
send(topic, message) -> None
read(topic, offset: Optional) -> message
```

producer 可能会buffer, compress, flush 根据configuration, 以节省network calls.

consumer 可以set 一个 optional 的offset 以读取之前的message.

### broker

#### persistent storage

#### TTL

#### partitions and replications 
