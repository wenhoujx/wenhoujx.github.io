---
layout: post
title:  "系统设计: unique id generator"
date:   2023-11-21 18:00:00 -0400
categories: system-design unique-id-generator
---

**问题**:

看了很多unique id generator的方案, 这里总结一下.

## 单机方案

如果生成的id 没有什么奇怪的不能被猜中的限制, 如果需要id是可以被sort的.

最简单的就是mysql 的 auto_increment,

```sql
CREATE TABLE id_table (
     id BIGINT NOT NULL AUTO_INCREMENT
);
```

BITINT 是8 bytes, 64 bits, 2^64 = 1.8e19, 18 quintillion, 18 billion billion.

每次需要新的id:

```sql
insert into id_table () values();
select last_insert_id() from id_table;
```

因为是单机 所以不需要考虑race condition, 例如有两个client 同时insert, 那么有可能拿到相同的id.

:question: 为什么不直接用uuid就好了. 说实话平时 programming 的时候常常使用uuid, 但是有以下的问题

- uuid 太大了. 128 bits, 16位, 就是16bytes.
- uuid 不能被sort, 因为是random的, 这在某些场景下是个优点, 有时候是个缺点.
  - 比如如果给我的每一个tweet生成ID, 这时候如果是可以sort, 那么我们就很很容易得按sort id, 而得到按时间顺序排序的tweets.
- uuid是random的 所以往往要test一下是否已经有了这个id. 虽然wiki上说uuid基本是不会collide, 但是因为实际使用的时候很多人会truncate, 所以还是要test一下.

在面试和实际使用的时候, 单机一个mysql 是single point of failure 如果需要scalable 同时还要high availability, 那么就需要分布式方案.

## 分布式方案: snowflake

snowflake是twitter开发的unique id的一个方案.

![snowflake-bits](/assets/images/snowflake.jpg){:centered}

- sign bit 是unused, 保留这样不会让不支持unsigned的db发生negative number出问题.
- timestamp 41大概是69年, 2^41 / 365 / 24 / 3600 / 1000 = 69.7 years 的所有milliseconds
- generator id, 这个只要保证id gen 的机器每台的这个数值不相同就好了, 可以是datacenter id, thread id, machine id 等等.
- sequence number, 12 bits, 4096, 2^12, 4096 ids per millisecond.

多台机器, 这样每一个机器的generator id 不同, 这样每一机器每个ms能生成4096个id, 也就是说每秒能生成4096 * 1000 = 4 million ids.

好处是:

1. 每个id 都是按时间顺序的, 可以sort, 因为时间也是这个id的一部分.
2. 多个机器, 每个机器都可以生成id, 也就是scalable.

## 分布式方案: 两台或者多台mysql

这里我们又用到了mysql的auto_increment, 但是这次是两台或者多台mysql. 同时每一个mysql的auto_increment的increment和offset都不同.

```sql

mysql> CREATE TABLE autoinc1
    -> (col INT NOT NULL AUTO_INCREMENT PRIMARY KEY);
SET @@auto_increment_increment=2;
SET @@auto_increment_offset=1; 
-- or set offset to 0 , to get 0, 2, 4 ... 
-- SET @@auto_increment_offset=1; 
```

如果有两台机器, 一个generate 奇数, 一个generate 偶数, 那么就可以保证不会collide. 同时如果有一个下线了, 另一个也可以继续生成id.

这个方案的好处是:

- 可以sort, 在同为偶数或者同为奇数的情况下, 可以按照id的大小来sort.
- 简单, mysql的management比较成熟, 也比较简单.
- mysql 如果给到足够的资源也是能够scalable的.

缺点之一是如果一开始定下2台机器, 后面想改成 3台, 会比较麻烦.

## 分布式方案: redis

redis 有incr 这个atomic operation, 可以用来生成id.

```sh
# Increment the value at the key "my_counter" by 1 and get the result
127.0.0.1:6379> INCR my_counter
(integer) 1
```

因为incr是atomic的, 所以不会有race condition. 同时redis也是一个很快的可以scale的service

## 分布式方案: pre-compte ids

如果需要生成的id猜不到. 我们可以提前生成好一些id, 然后存到一个db里面, 每次需要id的时候从这个db里面拿出来一个id.

常见的id的length是6 bytes, 有2^48 个可能的id.

- 我们可以提前生成一部分的id, 存在一个table中.
- 然后每一个server 需要id的时候通过transaction来拿出N个id, 同时把这些id写到used table 中.
- 把读取的id放在cache中, 如果有request, 就给一个.

这样就算server crash了, 那些id只是丢失了.
通过每次读取N个id, 减少对sql db的lock.

## 分布式: hash

我们可一个通过hash来生成id, 例如md5, sha1, sha256, sha512 等等.

如果生成的id太大, 可以truncate成8bytes. 比如md5是16bytes, 我们可以取一半.

缺点是不知道这个id是不是unique, 但是可以通过test 已有的id来判断.

特别是如果server已经有可以让用户自己input id的逻辑, 这有时候我们也需要测试用户的input是不是unique的.

这个方案的好处是:

- 简单, 不需要额外的db, 只需要hash function. 虽然可能需要deploy mysql 存取这些已经使用的id.
  - 即使这些id被用在了key-value store, 我们也许不想增加critical service 的load, 而是选择check 一个dedicated mysql db

缺点是:

- 不知道要test多少次才能保证这个id是unique的. 虽然通过数学计算 collision是很少见的, 但是还是要测试, 不然会导致数据的丢失和泄露.
- hash 需要计算, 会增加load.

## 总结

以上就是我简单的常用的unique id generator的方案. 欢迎大家讨论补充.
