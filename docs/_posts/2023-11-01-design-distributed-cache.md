---
layout: post
title:  "如何设计一个distributed cache"
date:   2023-11-01 15:43:00 -0400
categories: distributed-cache system-design cap
published: true
---

**问题**:

- 最近在练习system design, distributed cache是一个很经典的问题.

## 先问需求

第一步总是要问一些问题, 确定需求. 虽然这些需求并不能帮啥忙. :sob:

既然是distributed, 那就问问:

1. scalability, 随着client 和 data的增加, 怎么scale up.
   1. 然后就问出了大概 client 和 data 的estimate
2. CAP theorem
   1. 你要consistency 还是要availability. 不能又当又立的.
   2. 如果牺牲consistency, 那么有的时候client会读到stale data, 但是你可以保证你的data是eventual consistent 因为cache的data一般都会被evicted.
   3. 如果牺牲availability, 那么有的时候client会读不到data, 就需要去server直接读取, 但是至少能保证data是最新的.
3. performance, 如果我用了cache ,还不如直接去server读取, 那我要这个cache 干什么!
   1. 这里涉及到不是所有的cache都是为了latency的, 有的cache是因为去server读取compute很很贵, 或者会overwhelm server, 或者server会call 3rd party 然后被bill.
   2. performance 最重要的就是要keep it in ram.

问到了数据就要大概得估算一些事情, 来显示你的数学能力. 这就是所谓的back of the envelope calculation.

比如有1million client, 要存储 1TB的data, 要保证一定的throughput 和latency. 不过这些都不会改变最后的design的答案.

### cache api

这里我们假设cache api只支持 `put` `get`.

```sh
get(key) -> optional<value>
put(key,value) -> None
```

### cache eviction policy

因为是cache 而不是key-value store, 所以应该有一定的cache eviction, 什么时候entry会expire?

- 时间
  - 如果过了一定时间 TTL (time to live) 就要evict

- 空间
  - cache只能存储N个entry, 如果超过了N个, 就要evict. 要么evict最早插入的, 要么用LRUevict最近最少使用的.

当然这些也和你具体的design 没有啥大关系, 只是为了显示你的knowledge 很全面罢了.

## Highlevel 答案

一下就是我能想到的要考虑的设计问题.

### Scalability

- 为了performance,我们要把entry放在ram里, 但是一个机器基本是放不下的,
- 为了scalability, 随着data的增加, 我们需要更多的machine.

所以我们需要sharding, 把 key space 分成独立的 subset 给每一个server.

具体就是compute 一个`hash(key)` , 然后根据结果放入不同的cache server

### Consistent Hashing

最简单的hashing 是一个 `md5` 然后 `mod` 给不同的server, 但是缺点很明显, 就是如果增加server, 所有的key都需要rehash.

[consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing)是一种common solution. it minimize rehash when the number of server changes.

#### Virtual Node

consistent hashing 的缺点是他不能evenly split 整个key space, 这里用到virtual node的[改进](https://stackoverflow.com/questions/69841546/consistent-hashing-why-are-vnodes-a-thing).

### Hot Shard

sharding的缺点是如果有的key 特别的popular, 那么负责这个key的server就是 hot shard.
常用的解决方法是 replicas, 一般是有一个leader shard, 和很多follower shards, 他们都是replica, 不同的replica 在不同的server 甚至data region.  

- `get` call 由所有的shards 随机选择一个处理. 也可以就近处理
- `put` call 由`leader` 处理, 然后在copy 给followers

### Availability vs Consistency

#### pick availability

replicas 涉及一个availability 的问题, 如果算上leader 总共有 n 个replica, 那么最多能支持 任意的 `n-1` 个server failures. 再多failures, 要去server直接读了.

由`leader` 处理`put` ,  在 `leader` 和 `follower` sync的时候 `client` 可能会读到stale data, 这时候其实是`eventual consistency`. 更惨的可能是 `leader` 还没有来及sync就挂了, 这时候`put`的最新数据就丢失了. 只能等到eviction来重新更新数据.

#### pick consistency

如果选择 `consistency` 而不是`availability`, 最常见的方法是 保证至少一半以上的replicas都ack 最新的update, 之后再慢慢的sync, 而且client需要选择有最新数据的, 或者每次`get` 需要读至少一半的node 然后选择最frequent的value.  这时候`put` 和 `read` 就会变慢.

### leader election

如何选择`leader` shard.

1. 可以hardcode 一个file configuration, 某某shard 某某server是primary/leader
   1. 缺点是每次要改动都要重新deploy, 而且无法实时的应对leader failure
2. 更好的更复杂的方法是用configuration service 例如zookeeper 保管所有的server, shard information, zookeeper可以帮助你选举leader.

### client 如何知道哪个server有想要的key

有很多种方法.

1. 同样的 你可以hardcode 一个configuration 在client里, 有所有的server, shard info,
   1. client每次可以自己去算这个key 应该在哪个server.
   2. 但是每次改动都需要重新deploy 或者upgrade client
2. 也可以把这个configuration 放在一个s3 让所有的client 去poll
3. 用configuration service, zookeeper 实时的会知道所有的server, client从zookeeper 读取当前live的server, 然后计算哪个server有想要的key.
   1. zookeeper会实时的通过heartbeat的方式或者其他来检测有没有server 掉线, 如果有, 要重新选择leader, 而且client也会在下一次poll的时候得到这个新的信息.
4. client可以随便query一个cache server, 然后这个server会自动route这个request到正确的server.
   1. 这也需要每一个server知道当前live的server, 所以需要zookeeper

## 最后 :tada:

和interviewer可以扩展的问题.

- 如果把eviction去掉, 一个cache server 和一个key value store, 或者document store是差不多的.
  - 除了 kv store, doc store 常常需要考虑indexing 和 paging的问题.
- cache client是 read through cache 还是 cache aside 还是别的什么. [详见](https://hazelcast.com/blog/a-hitchhikers-guide-to-caching-patterns/)
- encryption 是一个可以讨论的话题.
- 如果有很多的cache server, 有的server可能会比较慢, 有的server可能会比较快, 怎么让client选择最快的server.
  - load balancer
- 如果一个server down 会发生什么, 他胡汉三又回来了会怎么样.
  - down了之后, zookeeper 会失去heartbeat, 从而知道他down 了, 然后开始重新再 replicas种选择leader. 如果leader down 太久, 回不来了, zookeeper 会选择一个新的server 做一个新的replicas.
  - 一旦回来了之后, 一般会做作为follower重新加入replicas,
  - 如果leader down的时候有没有sync的data, 很有可能这个data 会丢失, 因为ta是以follower的低姿态加入的.
  