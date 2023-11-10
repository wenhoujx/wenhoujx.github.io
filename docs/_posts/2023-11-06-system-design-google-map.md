---
layout: post
title:  "系统设计: Google Map"
date:   2023-11-10 00:06:00 -0400
categories: system-design google-map routing shortest-path 
published: true
---

**问题**:

今天突然想到如何系统设计 google map, 参考了一些资料, 自己总结一下.

## requirements

1. avaiability
2. 要快, 但不用real time, 如果等2-3秒 compute route 是可以接受的.
3. accuracy, 推荐的路线应该是最优的, 或者差不多.
4. scalability, google map 用户很多 还有很多企业用户, 比如小app的公司会直接用google maps 的 API.

## v0: 实现地图static image的功能

首先我们设计最简单的功能, 手机打开之后能看到一个地球的图, 有点像google earth. 这个系统有几个components:

- mobile client(s)
- CDN
- static Map server(s)
- load balancer

手机用户打开手机, 根据自己的当前location, 向 map server 发送请求, map server 返回一个图片, 然后手机显示这个图片. 如果CDN已经cache了这个图片, 就无需向 map server 发送请求了.

![static-map-serving](/assets/images/static-map-serving.jpg)

### 地图上的道路从哪里来?

1. 问政府要.
2. 从地图上扫描出来.
3. 问其他的GPS公司买, 比如garmin.
4. 通过用户的实时地址的多年积累, 可以用ML得到道路的信息. [之后会提到]

## v1: 实现地图上显示places的功能

我们不想要简单的地图, 我们想在这个地图上标记各种places, 比如

- 广告: 餐馆, 酒店, 麦当劳, 加油站, 超市.
- 公共事业: 政府, 图书馆, 学校, 医院, 消防站, 警察局, 著名景点.

首先我们需要一个DB来储存这些places的信息, 这些信息并不会快速的变动, 我们可以用SQL DB 或者是cassandra 来存储.

每个place都有一些data:

- name
- address
- geo location, lat lon
- category
- tags

这些 places 的信息可以实时的overlay在static map上, 也可以pre-compute 之后, 提前cache在CDN上. 甚至有的CDN支持dynamic content caching.

## v2: 实现搜索地址, 搜索附近的功能

我们可以用我们储存places的DB的功能来实现搜索, postgres 或者 cassandra 都有indexing的功能, 可有一些基本的geo search的动能.

我们也可以用elasticsearch 作为一个secondary index 来实现搜索功能. elasticsearch 还附带支持强大的 geo search 功能. 比如我们可以问elasticsearch 附近的中餐馆.

```json
{
  "query": {
    "bool": {
      "must": [
        { "terms": { "tags": ["chinese"] } },
        { "term": { "category": "restaurant" } }, 
        {
          "geo_distance": { 
            "distance": "10km", 
            "location": { "lat": <user_lat>, "lon": <user_lon> } } }
      ]
    }
  }
}
```

![place-search](/assets/images/search-places.jpg)

## v3: 跟踪收集用户的实时位置

这个设计 privacy 的问题, 我觉得只有在用户同意的情况下才能实现. 现在的iphone 对个人隐私还是挺注重的 👍.

在用户同意的情况下, 我们可以让手机的background task 不时的发一个定位信息:

- 如果用户在移动 就每 5s 发一次.
- 如果用户在静止 就每 30s 发一次, 甚至更久.

具体的 arch components 有:

- mobile client(s)
- kafka event queue 用来存储用户的位置信息
- kafka -> spark streaming
- data 到了spark 之后可以用来跑各种ML.

大量的用户数据可以用来支持以下的功能:

- 估计道路的拥堵程度, 从而实现实时的路线规划.
- 如果很多人都经过一条路, 可以推测这是一条路. (走的人多了就变成了路)
- 可以通过用户的速度和行动的灵活程度 和 有没有等红灯来推测用户是开车, 还是走路, 还是骑车, 还是bus
- 可以推测这个一条路是单行道还是双行道, 是不是高速.
- 可以提前预测大量用户涌向一个地方, 说明这个地方要变得拥挤了.
- 可以用来推荐 lively 的area.

![real-time-location-tracking](/assets/images/user-location-tracking.jpg)

## v4: 从A 到 B

这个是最有用, 最复杂的功能, 一般也是design的重头戏, 不过其实已经是设计到了 algo design 了. 而不是纯纯的system design.

首先我们已知的是:

1. 我们已经有所有可能的道路的信息了.
2. 我们有用户现在的信息 还有 用户想去的地方的信息. 即 A 和 B 的信息.

### 分割 map (partition, segments)

如果只是用lat/lon来代表地址, 并找出所有通过A, B的道路, 然后在找shortest path, 这其实是一个很costly的操作. 我们需要一个更快的方法. [TODO: 为什么?]

常见的方法是把地球分割成很多很多块, 每一块里面继续分割. 分割的方法有很多种, 常见的有:

- grid, quadnode, geohash 这些都是分成方块.
- hexagon, 这个是分成六边形. 我挺喜欢这个的, 详见 [uber h3](https://www.uber.com/blog/h3/).
- 分成三角形
- 其他不规则形状, 比如 [google s3](https://s2geometry.io/).

![segment-shape](/assets/images/hex-u3.jpg)

正方形或者长方形的缺点是:

1. 每一个正方形的上下左右的邻居和nw, sw, ne, se的邻居 到中心的距离不一样, 这样不能快速的通过segment来估算距离.
2. 正方形 在高纬度的地方distort 严重, 会cover更大的一块区域, 导致每个level的正方形的size 也不一样.

不过正方形还是别的什么形状, 不会影响到这里的设计.

### 每一个segment 里面的道路信息

对于每一个segment, 我们知道这个segment里面的所有道路的信息. 更重要的是 我们知道所有的exits, 即和其他segment相连的道路的信息.

![segment-exits](/assets/images/within-a-segment.jpeg){:width="50%"}

### build a graph

从 A B两点开始, 可以用BFS找到一片 segments 把这两个点相连. 如果B在A的东边, 那么我们还要包括A西边的N个segments 还有 B 东边的 N 个segments, 以防有一条更好的路在这些segments 里面.

我们要找的路大概率就在这一片segments里面, 当然也会有漏网之鱼, 这和我们的 N configuration 的设定有关. N 越大, segments 越多, 漏网之鱼越少, 但是搜索的时间越长.

我们可以用这一片segments来build一个graph:

- 通过每一个segment 和邻居的segments的exits的相连, 我们得到了一个精简的graph.
  - 这一步还排除了一些segment, 因为他们没有参与到最终的graph里面. e.g. 有一条河把一片segments分出去了.
- 有了这个graph之后我们需要compute edge weights.
  - 通过graph, 我们知道了对于一个segment, 哪些exits 参与了这个graph
  - 对于这个同一个segment 的exits, 我们计算他们的weight.
    - 这里的weight 可以是distance, 可以是ETA, ECO score, 也可以是他们的combination.
    - 如果这个segment很大, (比如我们要从纽约开去LA), 我们可以进一步把这个segment分成更小的segments, 然后用同样的方法计算exits的weight. 直到segment小到好计算为止.
    - 越小的segment 计算起来越快, 因为routes的可能少.
- 有了graph 和 edge weights 之后我们就可以用shortest path, 来计算从A到B的最好的路线了.

### 优化: contraction hierarchies

Precompute 一个segment 的 segment 的最优路线, 一般都是一些主要的highway. 这个优化的原理是公路是很hierarchical的, 一般都是主干道, 还有一些分支, 分支在很多时候都很难是最优的路线. 如果我们提前算了最优路线, 那么我们以后就可以直接用这些最优路线.

Precompute在一个segment内部从exit到另一个exit的最优路线.

### 优化: cache 最优路线

在一定时间内, 从A到B的最优路线是不变, 我们可以cache计算结果, 如果有人再问, 我们就直接返回这个结果.

### 优化: cache historical 最优路线

例子: cache 下每周一(如果不是holiday) 下午4点的 每个segment的weight 和precompute的最优路线, 这样我们大概率可以重复使用.

## traffic, road closure, accidents

这是一个很头疼的问题. 如果没有这些condition, 我们可以precompute很多东西, 之后直接拿来用. 但是如果有这些改变路线 weight 的情况发生. 我们需要实时的观测, 改变 edge weight, 然后pop up 到更大的segment上, 然后更新那些路段的 weight, 然后重新compute最优路线.

这里也可以就算这些condition 造成的最优路线weight的变化, 只有在超过一定 threshold 的时候才会trigger 重新寻找其他最优的路线.

## 放在一起, arch

把这些优化放在一起, 可以得到这样的arch:

![compute-route-arch](/assets/images/compute-route.jpg)

## links

- [ETA Phone Home: How Uber Engineers an Efficient Route](https://www.uber.com/blog/engineering-routing-engine/)
- [Contraction hierarchies](https://en.wikipedia.org/wiki/Contraction_hierarchies)
- [uber u3](https://www.uber.com/blog/h3/)
