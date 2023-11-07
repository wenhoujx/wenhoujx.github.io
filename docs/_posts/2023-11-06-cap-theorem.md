---
layout: post
title:  "CAP theorem到底在说什么"
date:   2023-11-06 16:28:00 -0400
categories: cap-theorem system-design distributed-system replicas
published: true
---

**问题**:

- [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) 长期霸榜distributed system design, 但是他到底在说什么?
- 这篇用尽量通俗易懂的方式讲清楚.

## CAP

![CAP venn diagram](/assets/images/cap.png){:width="50%"}

- C `consistency`
- A `availability`
- P `partition tolerance`

只能选两个.

首先如果是design一个distributed system, `P - partition tolerance`是必选的, 有很多情况会触发 network partition:

- network failure, delay, network down, 或者被人拔线了.
- server failure, 或者garbage collection 花了太长时间, 或者死循环了, 这种情况下, 其他的server很久都没有听到他的消息, 就会自动认为他down了.

P 其实可以被理解为是distributed system 必选的, 所以只能选择 C 或者 A 剩下的一个, 所以才会有 CP system 和 AP system.

## CP systems

选择 Consistency, 意味着 所有的 acknowledged writes 都会被之后的read读到, 而不会读到stale data. 注意如果被`ack`的`writes`不算, 那是`no availability`的情况.

一般都是银行或者金融系统 选择CP. 这些系统牺牲availability, 意味着如果`network partition` 发生了, 情愿system down (unavailable), 也不要去强行操作.

不然之后一定会有 data consistency 问题, 有的还是不能自动resolve的, 具体表现就是我的银行了多了1w美金 🎉 , 或者少了 😢.

一个例子就是如果有两台`ATM` 他们的data 是通过网络`sync`的, 这时候如果一台ATM检测到另一台消失了, `network partition` 发生了,

- 这时候ATM如果选择了 `availability`, 那你还能存钱取钱. 但是有可能有人会从两台ATM中都把自己的毕生积蓄取走了, 直接财富翻倍 💰.
- 所以银行自然都会选择 `consistency`.

如果你找到了 AP系统的银行, 一定要告诉我.

## AP systems

选择 availability, 意味着即使`network partition`发生了, 任何一个server都能serve requests 如常, 注意是任何一个server.

这里不考虑server failure, 只考虑真正的network partition.

举个例子

- 最坏的情况是变成了一个 cluster 变成了两个clusters,
- 比如你有网上血拼的时候有一个购物车.
- 你加入了一把菜刀 🔪, 这个request 给到了cluster1, 他记下了.
- 你又加入了一个切菜板, 这个request给到了cluster2, 他也记下了.
- 这时候如果你刷新, 很有可能菜刀或者切菜板不见了.
- 如果network 修好了, 有的system会二选一, 有的system会merge到一起.

银行是不会选择AP的.

E-commerce或者hotel booking会选择AP, 因为他们情愿用户有好的用户体验, 之后在人工resolve consistency, 也不愿意用户一直看到down的页面.

## 成年人不做选择

![i want them all](/assets/images/i-want-it-all.gif){:style="display:block; margin-left:auto; margin-right:auto"}

市面上大多的系统都是三个都要, 但是其实就是都不要.

各种DB标榜自己怎么怎么durable, 怎么怎么available, 怎么怎么不会丢失数据, 其实都是狗屁 🐶 💩, 因为是不可能的. 详见[jepsen](https://jepsen.io/)大神对各种DB的测试, 最终的结果总是让人失望.

其实很好理解为什么CA不能兼得, 虽然有严格的证明, 但是我来用我实际的经验来解释一下.

 我们考虑以下情况:

- 在一个distributed system有两个server A和B.
- 首先考虑没有data replicas的情况, A和B都有自己**独自负责的data**. 这个系统有single point of failure.
  - 这时候如果 `network partition` 发生了, A和B不能交流了. 这时候如果有request进来问B只有A知道的data, 那B要么老老实实说自己不知道(no availability), 要么就随便给一个值 (no consistency).
  - 同样的情况, 如果是有request进给B 要update一个只有A才知道的值, 那么B要么说自己不知道(no availability), 要么B把这个值写下来, 但是这个值不会被replicate到A, 如果之后有问A read 这个值的request, 那就只能有stale data (no consistency).
- 不replicate data意味着如果A或者B server crash了, 就没有了data, 更常见的是A和 B replicate data and in sync的情况, 两个都有一样的data.
  - 如果这时候 `network partition` 发生了.
  - A和B这时有同样的data, 但是他们之间不能交流了.
  - 任何的write request, 要么A 或者 B 说自己不能操作(no availability), 因为就算记下了也没有办法告诉他的兄弟.
  - 要么其中一个server收下了这个write, 但是不能sync到另一个server, 那就会有consistency issue.

### 好消息

好消息是其实CA之间不是只能binary choice的, 而是可以通过configuration来取中间的.

问题是这就给了很多DB玩发明新词汇的可能性, 这就是为什么市面上很多DB说自己又C又A的, 其实都是玩弄文字罢了. 比如instead of consistency, 发明出一个eventual consistency, 比如availability, 发明出一个high availability.

其实都是选择了C和A之间的configuration. 所以在选择DB的时候一定要彻底理解这些概念, 不要被他们的marketing忽悠了.

这里举个例子, 很多db比如 `mongodb` 和 `kafka` 都可以configure 每个write request需要多少个server acks(确认接收). 通过configure这个值可以控制C和A之间的tradeoff.

比如你有n个servers, 为了保证data还是available如果真的有server down, 你shard了data, 而且每个shard总共有r个replicas, 这些都是基本操作.

- 情况1:
  - 为了write speed, 你要求write只需要一个server ack就好了(mongo default), 之后这个leader server会把这个write sync到其他的server, 然后read可以从任何一个ISR (in sync replicas) 读取数据. 这是一个经典的async replication的setup.
  - 如果选择这个configuration 其实是牺牲了consistency, 得到了fast write和availability.
  - 如果leader down, 那么所有的没有sync的data都很有可能lost了, 如果选了新的leader, 之后旧的leader重新上线, 那么他们之间的data就会不一样, 这时候就会有consistency issue.
- 如果你选择另一个极端, 需要所有的r个replicas 都ack. 当然你的write 就会很慢, 而且如果某一个replica server down了, 你个任何write 这个shard的request都会失败, 因为没有r个replica了. 系统只能慢慢的挪shard 创建一个新的replica, 但是期间是没有availability的.
  - 但是你永远都不会有inconsistent data, 因为所有的replica都有一样的data.
- 如果你选择中间, 常见的是quorum, `r // 2 + 1` (一半以上的replicas) 个write acks.
  - write request最多支持 `r//2 -1` 个nodes down, 同时还能保证data consistency. 但是一定要保证request不要被route到这些down的nodes. 因为他们是不能serve request的, 他们没有 write quorum.
  - read request 也最多支持 `r//2-1` 个nodes partition, 保证data consistency, 其实就是整个cluster从n个nodes 变成了 `n-(r//2 -1)` 个nodes 同时支持read和write.
  - 这时其实是没有strong availability, 因为不是所有的server都能serve request. 可以用一些方法保证被partition出去的nodes不会serve request, 常见的比如在至少有一半以上的quorum中进行leader election, 因为被partition出去的nodes不能形成一半以上的quorum, 所以他们选不出leader, 所以不会变成available.

## 总结

CAP 指导我们design distributed system:

1. 你可以选择CP 或者 AP system.
   - 金融选CP
   - E-commerce 选AP
2. 或者你可以取中间.
   - 大部分的db都是这样的. 其实是把问题抛给了用户, 让用户自己选择configuration.
3. 选择市面上的db的时候一定要理解他们的failure mode, 不然就会被他们的marketing忽悠了.
