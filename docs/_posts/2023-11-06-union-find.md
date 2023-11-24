---
layout: post
title:  "用union find给元素快速分组"
date:   2023-11-06 12:28:00 -0400
categories: algo data-structure union-find
published: true
---

**问题**:

- 当你有一堆东西, 需要根据某种条件分组, 有什么data structure可以用?
- union-find 是什么
- 我每每刷leetcode的时候经常遇到需要这种data structure, 每次都另辟蹊径的解决了, 知道现在我才发现有一个叫union-find的data structure可以用.

## 举例

如果有一堆东西, 我们用`[1,2,3,4,5]`来代表. 我们iterate through的同时要根据某些条件给他们分组, 如`[1,3,4]` 一组, `[2,5]`一组. 这时候我们需要一个data structure来存储这种grouping.

**具体的例子**:
比如

## union-find

`union-find` 是一个data structure, 有两个主要的API:

```python
union(a,b) # 把a,b放在同一个group里
find(a) # 找到a所在的group
```

其中 `find` 也有把元素`a`加入data structure的作用.

这个data-structure的要点是

- 在每一个组中选择一个 `root` 爸爸, 这个爸爸element就是`find` return的值.
- 同一个group中的元素形成一个`tree`, 这个tree最上面的元素root就是这个爸爸.
- `root` 爸爸可以有1-n个孩子, 每个孩子也可以有他们自己的孩子. 如果我们call `find` on 任何一个group里的元素, 都会返回root爸爸.

比如以上的例子, `[1,3,4]` 的爸爸可以是1, 也可以是3, 也可以是4. 但是只能是一个. 这和具体的implementation 和insert order有关系.

比如我们选择了 1 是爸爸, 这时这个`tree`的一个可能的样子是 (其实没有left, right孩子的区分)

```sh
1   # 1 的孩子是3    find(1) = 1
 - 3  # 3 的孩子是4  find(3) = 1 
   - 4  # 4 没有孩子 find(4) = 1

2  # 2是root爸爸 的孩子是5    find(2) = 2
  - 5  # 5 没有孩子           find(5) = 2
```

理解了这些就大概知道`find`是一个`recursive`找root爸爸的过程了, 但是`union`是怎么样的呢.

`union`其实就是demote一个root爸爸, 把一个group的root 爸爸变成另一个group的root爸爸的儿子. 就像是两个家族找到了共同的祖先, 他们就成了一个家族.

### 具体实现

一旦理解了内部的结构, 这个data-structure就很好实现了.

```python
class UnionFind(): 
    def __init__(self): 
        self.par = {} # 这个dict 从孩子指向爸爸

    def find(self, el): 
      # find root 爸爸, 同时也有把元素加入到这个data structure的作用
        p = self.par.get(el) 
        if p is None :  # 如果是一个新元素, 那么他就是自己的爸爸, 他自成一派
            self.par[el] = el 
            return el 
        elif el == self.par.get(el): # 如果自己是自己的爸爸, 那他就是root
            return el 
        else: 
            return self.find(p) # 如果有爸爸, 那就找到他爸爸的可能的爸爸.

    def union(self, a,b): 
        # 这里a,b可以互换位置, 重要的是把一个root爸爸变成另一个root爸爸的孩子
        # 如果a,b本来就是同根生, 那么这个操作是no op. 
        self.par[self.find(b)] = self.find(a) 
```

上面的[1,2,3,4,5]的例子, 我们可以建立这样的union-find:

```python
uf = UnionFind()
uf.union(1,3)
uf.union(3,4)
uf.union(2,5)
```

最后常常要找到一共有几个group:
  
```python
len(set(uf.find(el) for el in uf.par))
```

### leetcode 例子

- [959. Regions Cut By Slashes](https://leetcode.com/problems/regions-cut-by-slashes/solutions/3032214/python-solution-union-find-with-4-regions/)
- [1584. Min Cost to Connect All Points](https://leetcode.com/problems/min-cost-to-connect-all-points/)

## complexity

### time

- `find` 是一个recursive的过程, worst case是O(n), 但是average case是O(1)
- `union` 用了`find` 所以worst是`o(n)`, average是O(1)

### space

- O(n) 是`self.par`的大小
