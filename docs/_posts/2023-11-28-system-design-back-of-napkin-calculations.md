---
layout: post
title:  "系统设计: 估算"
date:   2023-11-28 18:00:00 -0400
categories: system-design quick-calculation
published: true
---

这篇是总结各种系统设计中的快速估算的cases.

## 为什么要估算

我之前很瞧不起系统设计估算这件事情, 觉得估算对后续的design并没有什么实际的意义, 因为每次还是要设计一个big distributed system. 并不会被估算的事情改变.

后来我改变了想法:

1. 首先估算这些东西本来就是面试时候考察的一部分, 测试你的西靠能力的系统性还有全面.
2. 之后估算确实会改变之后的设计, 如果算出来特别大就要考虑sharding, 如果没有那么大 一个机器也能搞定, 就可以通过master-slave replica 系统来保证high availability 还有read throughput.

## 估算的方法

基本需要通过和面试官的对话假设一些数据, 最重要的就是:

- MAU, DAU, 月活跃用户, 日活跃用户.
- 系统中的各个entities之间的联系. 比如每个人follow 多少人, 每个人每天发帖数多少, 每用户每个月request 多少uber rides
  - 这里的估算可以尽可能的保守, 因为每一个系统都大多数都是不那么活跃的用户.
- request byte size, response byte size, 这个可以通过把data model设计的fields加起来来估算.
- 有些services在peak hours会有burst  traffic, 我们可以假设3倍甚至是10倍的traffic.

有了这些假设就可以估算出来以下几个打的categories:

- QPS (queries per second)
- storage
- memory
- inbound, outbound bandwidth

### 估算的利器

- 20-80 rule, 比如cache的size, 我们可以argue 在最近的一个小时中20%的data占用了80%的hit request. 所以cache的size是总的outbound bandwidth的20%.
- 1million / 86400 = 11.5 这个计算很多时候可以用到, 比如有每天有1bil 的request, 那么就是11.5k request per second.
- 既然是估算, 不需要精确, 9 可以约等于10. 注意数量级不要搞错就好了.
- `2^10 == 1024 ~= 1000`, 所以估算8bytes能代表多少个数字的时候, 可以用`2^32 ~= 4 billion`来估算.

## 估算的例子

以下整理归纳自己的估算的例子.

### rate-limiter

假设

- 100MIL DAU, 每个用户产生100个requests
- burst hours 3倍traffic
  - 我们可以强调系统需要支持 auto scalable, 而不需要人为的干预.
- 假设每个request 是一个key, 8bytes. 每一个response是一个boolean value 1bit 忽略不计.

推导出:

- QPS: `100mil users per day / 86400 seconds per day = 1.15k QPS.`
- 3倍burst traffic: `3 * 1.15k = 3.45k QPS`
- inbound bandwidth: `8bytes * 1.15k = 10k bytes per second`
- outbound bandwidth 忽略不计.
- 如果我们把过去2小时的request都放在memory的cache中, 假设每一个key背后都对应这一个counter `2bytes (2**16 ~64K),` 那我们需要的memory是 `1.15k requests per second *2 hours* 3600 seconds per hour * (2 + 8) bytes = 82GB`.

这里我们可以假设过去2小时的request只有20% unique的keys, 那么我们需要的memory是 82GB * 20% = 16GB. 突然就能全部fit在一个machine的memory中了. 这样我们就能假设用一个master, 很多个slave:

- master更新token limit, slave用来async update.
- master和slave都可以用来 serve read requests.

当然就算不能fit一个machine中,我们也可以讨论sharding的方案. 把每个request按照request key shard到不同的machine上, 由这个machine来serve这个request. 然后就能引出一些sharding的问题, 比如:

- hot uneven shard.

然后解决方案依然是套路的:

- consistent hashing
- virtual nodes

### yelp

假设:

- 500MIL places, 100MIL DAU.
- each user searches twice, visited 5 places per day.
- assume each place has `(id 8bytes, lat 8bytes, lon 8bytes, name 20bytes, bio 500bytes...) -> roughly 600bytes`
- assume each search request has `(user_id 8bytes, lat 8bytes, lon 8bytes, radius 8bytes, keyword 20bytes) -> roughly 50bytes`
- assume each search returns 10 places

推导:

- search QPS: `100mil DAU *2 searches per day / 86400 seconds per day = 11.5* 100 * 2 = 2.3k QPS`
- fetch place QPS: `100MIL DAU * 5 places per day / 86400 seconds per day = 5.8k QPS`
- inbound bandwidth: `2.3k *50bytes + 5.8k* 600bytes = 3.5MB/s`
- outbound bandwidth: `2.3k *10* 600bytes = 13.8MB/s`
- storage: `500mil * 600bytes = 300GB`
- memory: assuming keeping 1 hour outbound bandwidth in memory, `13.8MB/s * 3600 seconds per hour = 50GB`
- or assume keeping 20% of places in memory, `300GB * 20% = 60GB`

如果implementation用到`quadtree`, 假设每一个leave node keep 100 places, 那么我们有 `500mil / 100 = 5mil`. 大概有`1/3` 的internal nodes, 每一个node有2个pointer (8byte each), 那么就是 `5mil / 3 * 2 * 8bytes = 26MB`, 加上leave中存储了500MIL place 总共还是 `300GB`.

当然我们可以只在leave node中存储place的id, 然后在ram或者db 或者disk中存储place的信息. 这样就可以减少memory的使用.

### top k service

假设

- 假设整个space有1bil的key, 每天有1bil的request
- 每一个request, return top 10 keys

推导:

- QPS: `1bil / 86400 = 11.5k QPS`, 大概是一个machine可以handle的, 或者2-3个.
- outbound hits per second: `11.5k inbound QPS * 10 returned items per request = 115k`

如果我们用CMS (count-min sketch), 选择 `width = 1mil, height = 1k` (hash functions), 每一个counter是`4bytes (2*32 ~= 4bil)`

- storage: `1mil *1k* 4bytes = 4GB`

### autocomplete

假设

- 1bil DAU, each searches 1 times per day
- each search has length 15 bytes, we trigger about 5 autocomplete request per search
- search requests are lightweight, 10 bytes roughly on average.
- each search hits 10 autocomplete results, each result is 15 bytes roughly.

推导:

- QPS: `1bil * 1 searches per day / 86400 seconds per day = 11.5k QPS`
  - 绝对不是一个machine可以handle的.
- outbound bandwidth: `11.5k * 1 searches * 5 autocompletes * 10 results * 15 bytes = 8.6MB/s`
- keep past 1 hour response in ram, consider 20-80, assuming only 50% unique: `8.6mb/s * 3600seconds = 31GB`

假设我们用trie, 最开始的node有52中选择(26小写+ 26大写 + 10digits) 每一个node大概有10个popular children, 平均search是15bytes. 那么就有: `52 * 10^14 = 520trillion`  个node, 平均每个node keep top 10 items, 每个15bytes, 那么就是 `520trillion * 10 * 15bytes = 7.8PB`

### newsfeed service

假设

- 1bil DAU, each user follows 100 people.
- each user login 1 times per day, each user fetch 10 times their newsfeed per day.
- each newsfeed has `(id 8bytes, content 500bytes, author 8bytes, timestamp 8bytes, pointers to media:24bytes) -> roughly 550bytes`

推导:

- QPS: `1bil * 1 login per day * 10 fetches / 86400 seconds per day = 115k QPS`  
- outbound bandwidth: `115k * 10 * 550bytes = 632MB/s`
- storage `1bil * 1 * 550bytes = 0.55TB` per day
- cache the past hour in ram, consider 20-80, assuming only 20% hot tweets serves most requests: `632MB per second* 20% * 3600 = 4.5TB`

### messaging service

假设:

- 1 bil DAU, each user sends 2 message per day, each message has 3 recipients.
- 每个message有500bytes.
- 1/5的message有image (1mb), 1/20的message有video (10mb)

推导:

- QPS: `1bil * 2 messages per day / 86400 seconds per day = 23k QPS`
- outbound bandwidth: `23k * 500bytes + 23k * 1/5 * 1mb + 23k * 1/20 * 10mb = 1.2GB/s`
- 5 times burst: `1.2GB/s * 5 = 6 GB/s`
- storage: `1bil * 2 * 500bytes = 1TB` per day
- memory, handle 1 minutes of inbound + outbound bandwidth: `1.2GB/s + 6GB/s * 60 seconds= 360GB`

### url shortener

假设

- 100mil DAU, each user generates 0.5 short url per day.
- 20:1 read:write ratio
- 每一个short url 有8bytes, 每一个long url 有200bytes

推导:

- QPS: `100mil * 0.5 short url per day / 86400 seconds per day = 600 QPS`
- storage: `600 QPS * (208bytes) * 86400 seconds per day = 10GB` per day
- 50mil requests per day in memory, `50mil * 208bytes = 10GB`. fits one machine.

### web crawler

假设

- 40bil page, 每个html page有500kb.
- 每个月都要重新check一遍所有的page.
- 每个page有5个links

推导:

- storage: `40bil * 500kb = 20PB`
- QPS: `40bil / 30 days / 86400 seconds per day = 15k QPS`
- download bandwidth: `15k * 500kb = 7.5GB/s`
- keep each page in ram for 10seconds for processing: `7.5GB/s * 10 seconds = 75TB`

还要考虑很多page是duplicated的. 其实是为了展示自己的思想的全面.

## 总结

这篇总结系统设计的估算的套路, 希望对大家有所帮助, 如果有什么意见问题欢迎留言.
