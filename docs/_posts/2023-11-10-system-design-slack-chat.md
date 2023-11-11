---
layout: post
title:  "系统设计: Slack Chat"
date:   2023-11-10 00:06:00 -0400
categories: system-design google-map routing shortest-path 
published: true
---

**问题**:

如何系统设计 slack chat.

## requirements

1. availability
2. latency, 要在一秒内load消息,
3. 这里focus在chat, notification 上,  而不是其他的功能, 比如video chat, screen share, file share, etc.
4. scalability 要支持很多用户

## v0: 用户profile

我们需要一个database 存储一些基本的objects, 并且能够搜索他们.

- 用户的profile
- workspace, 用户可以被加入workspace.
- channel, 每个workspace里面有不少channel,
  - 这里假设用户可以搜索所有的public channels.
  - 用户可以被加入 channel, 这里不考虑踢出.

假设一个用户的基本信息是一个json, 大概有1000个char, 那么就是1kb, 我们保守估计有10kb, 如果有1billion 用户, 就是10TB data, 这样的大的data存储有两个options:

1. horizontally scalable sql db, 比如vitess, redshift.
2. nosql document store, 比如cassandra, mongodb, scylladb.

这些db都有indexing 功能, 可以用于快速搜索用户 或者namespace 或者channel.

![slack object store arch](/assets/images/slack-object-store.jpg){:width="70%"}{:.centered}

### 基本的api 如下

```python
create_user(user_name, {other info}) -> user_id
create_workspace(workspace_name, ...) -> workspace_id 
create_channel(workspace_id, channel_name) -> channel_id 

add_user_to_workspace(workspace_id, user_id)
add_user_to_channel(channel_id, user_id) # possible fail if user_id not in workspace 

search_user(...) -> List[user_id] # possible paging 
search_workspace(...) -> List[workspace_id] # possible paging 
search_channel(...) -> List[channel_id] # possible paging 

get_users_in_channel(channel_id) -> List[user_id] # possible paging
get_channels_in_workspace(workspace_id) -> List[channel_id] # possible paging
get_users_in_workspace(workspace_id) -> List[user_id] # possible paging

```

## v1, 重头: 设计收发message

### choice of message store

slack用Vitess scale MySql, Shard所有的message by workspace_id. discord 是另一个很流行的chat app, 他们用的是cassandra, 不过2017年他们migrate到了一个cassandra compatible的nosql store, scylladb.

所以我们可以看到在存储海量的message chat的数据的时候, 并没有一个统一的solution, 往往是受一下的因素影响:

- 一开始的dev team 的选择.
- 整个团队的operational expertise.

### MySql, Vitess, Sharding

如果使用的是sql db, 那就一定要考虑horizonal scaling的问题, Vitess就是应运而生的科技.

Slack 选择是的shard by `workspace_id`, 即所有的workspace的message都在一个shard及其replicas中, 这样简化了debugging和一些performance, 因为能够利用data locality. 每个shard都至少有一个replica (有几个replica 要看这个enterprise付了多少钱) 是在别的datacenter的, 这样如果一个down了, 也能有availability. 中途丢失一些数据也是可以接受的.

但缺点也很明显:

- hot shard, 假设整个trump的粉们都在一个workspace, 那么这个workspace就会很hot, 而且充斥着各种谎言以及歧视. 🐶.
- too big a shard, 如果apple明天要migrate到 slack上, 10年之后, 这些message 有可能超过一个shard能放的, 这时候就需要vertically scale 这个shard.

slack 在一个blog 中透露后悔没有当时shard by `channel_id`. 这里的lesson是要遇见以后的数据量, shard的时候可能要更加fine grain一些, 但是也不能走极端, 比如每个message都单独shard.

如果单独shard每一个message, 那么load是even 了, 但是想想每次load一个channel的message的时候, 或者search workspace message的时候都会fire up 所有的shards, 那么多的disk page seek miss, 然后还有耗费在coordinate node上的sort 还有aggregate, 也会是很慢的, 甚至会有GC issue.

chat app有一个天生的有点就是message 在一定时间之后就是immutable的了, 不会需要用到transaction 去atomically update很多message的情况. 我们都知道 distributed systems performance 最大的敌人就是locks.

### NoSql approach

如果像discord一样用cassandra 或者是 scylladb,

discord不用cassandra的原因是 cassandra的read比write 要expensive! 因为cassandra要query有所的on-disk SSTalbes (LSM). 而write只需要写到一个SSTable. 可以通过aggressive merge LSM来降低read的成本, 但是merge LSM也是一个很expensive的操作. chat app 本来就是一个read比write多多的系统. 而且如果一个partition的read overwhelm了这个partition, 那么所有这个partition支持的channel loading都会变慢.

使用NoSql 也需要design sharding (partition keys), 也需要考虑hot shard/partition的问题. 很多时候使用sql 或者 nosql其实都差不多. 只是要确切的了解db的read/write pattern 以及是否需要sql提供的transaction, nosql 提供的flexible schema.

discord使用了一caching, coalescing tech 在他们的server 和 db 之间. 如果很多discord server都发送同样的request, coalescer 会只query db一次. 这种情况常常在有人 `@everyone` 的时候发生.

[🌶️ 吐槽一下, 如果不同的人有不同的读取message的权限, 往往会ruin caching 或者这种coalescer]

![discord-query-coalescer](/assets/images/discord-query-coalescer.jpg)

### 设计接受message的api

我们选择了message store db之后就要考虑用户发送消息的流程以及system的设计.

我们用以下的例子:

```python
send_message(workspace_id, channel_id, {
  message_body: ... , 
  author_id: ... , 
  pinged: [...], 
})
```

系统的设计如下:

- client通过websocket连接server,
  - websocket因为2way communication, 而且快.
- 如果是write/update message, 直接route到kafka中,
  - kafak因为可以handle很高的write ingestion, 而且message 是durable的, 以防之后要replay.
- streaming indexer 读取kafka, 快速的index到 message db中.
- 用户可以query slack server读取message, 或者query workspace, channel.
  - searchability 可以是db自己提供的也可以用elasticsearch 做一个secondary index.
    - 要注意的一点: maintain这么大的es cluster是一件很耗费人力的事情, 最好还是用db自带的indexing来manage search/query. 这样可以减少operational cost 和 training cost.

![slack-messenger](/assets/images/slack-messenger.jpg)

## v2, 设计notifications

根据自己的notification 设定, 我们可能会被ping到有新的message. 比如有人 `@everyone`的时候, 所有在这个channel里面的人都会被 ping 到.

message 一进来就会被queue到kafka里面, 我们可以deploy 一个job, read message, 然后决定谁被 ping了. 我们可以在客户端就parse message, 决定谁被 ping, 这样server端就不需要parse这个信息.

知道一个message ping 了一些users, 然后我们怎么办呢.

### option 1: 把notification event 放在kafka queue里面

最简单的是放进kafka里面, 给每一个用户都设置一个queue, 里面是他们的未读消息, 从就到新. 这样重新利用了我们的kafka queue, kakfa 在很多topic 下也能operate, 但是这个设计有一个重大的缺点, 我们没有办法efficiently delete一个已读信息out of the queue.

为了支持已读信息, 我们还是用回我们已有的db来支持CRUD.

一个notification message的body 大概如下:

```json
{
  "type": "notification", 
  "message_id": "123",
  "channel_id": "456",
  "workspace_id": "789",
  "author_id": "101112",
  "read": false
}
```

### option 2: 把notification 放回db

这时候我们可以选择把这个message 放在db中. 然后create index by user_id, read, 我们可以query用户未读的notification, 然后全部发给user, 如果user读了某一条, 我们可以`mark read == True`, 然后deploy 一个task删除已读已读的信息.

如果每次有新的信息, 我们都issue 一个 count query, 这也是不efficient的, 我们可以用redis cache 当前一个用户的未读, 然后每次 +1 , deploy 一个task每隔一段时间就更新这个count以保证accurate.

具体系统设计如下:

![slack-notification](/assets/images/slack-notification.jpg)

### option 3: put notification in db as a list under user

我们可以设计如下的user document:

```json
{
  "user_id": "123",
  "notifications": ["message_id", ...]
}
```

每次有新的notification, 我们都加入到这个document的 `notifications` list里面, 这里的tradeoff是写会变得更加costly, 但是读只需要读取用户的`user_id` 就可以了, 不需要再去query notification db. 具体采用那个, 其实都不是interview的重点, 重点是说出这些tradeoff.

## 总结

我们介绍了如何design 一个 chat app, 这里用slack为例子.

这个system design 还能用在任何的messenger app, 比如facebook messenger, whatsapp, wechat, etc. 甚至是notification service for tweeter(X). 

## links

- [slack scale message store with Vitess](https://slack.engineering/scaling-datastores-at-slack-with-vitess/)
- [how-discord-stores-trillions-of-messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)
