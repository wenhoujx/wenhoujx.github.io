---
layout: post
title:  "Kadane algo 解释"
date:   2023-12-07 18:00:00 -0400
categories: algo dynamic-programming kadane 
published: true
---

Kadane algo 用于找到一个array中的最大的subarray sum.
我之前只是死记硬背的记住了这个algo:

```python
def kadane_algorithm(arr):
    max_current = max_global = arr[0]

    for i in range(1, len(arr)):
        max_current = max(arr[i], max_current + arr[i])
        max_global = max(max_global, max_current)

    return max_global
```

之后做题的时候也总是可能写错, 一阵子不就会忘记.

但是慢慢的也对这个algo有了一定理解, 知道今天才发现这algo其实是一个dynamic programming (DP) 的algo.

# 解释

首先我们不要去管 `max_globa` 这个variable, algo就变成了:

```python
def kadane_algorithm(arr):
    max_current = arr[0]

    for i in range(1, len(arr)):
        max_current = max(arr[i], max_current + arr[i])

    return max_current
```

如果我们把每个index的`max_current` 记录在一个array `save` 之中:

```python
def kadane_algorithm(arr):
    max_current = arr[0]
    save = [0] * len(arr)
    save[0] = max_current

    for i in range(1, len(arr)):
        max_current = max(arr[i], max_current + arr[i])
        save[i] = max_current

    return max(max_current)
```

这个时候我们就发现其实我们不需要 `max_current` 这个variable, 因为我们只需要知道 `save[i-1]` 就行了. 我们把`save`改名为`dp`.

```python
def kadane_algorithm(arr):
    dp = [0] * len(arr)
    dp[0] = arr[0]

    for i in range(1, len(arr)):
        dp[i] = max(arr[i], dp[i-1] + arr[i])

    return max(dp)
```

这时候这个algo在做什么就变得清晰了. `dp[i]` 就是以`arr[i]`结尾的最大的subarray sum.

如果以`i-1` 结尾的最大的subarray sum是负数, 那么`dp[i]` 就是`arr[i]`本身. 如果以`i-1` 结尾的最大的subarray sum是正数, 那么`dp[i]` 就是 `arr[i] + dp[i-1]`.

如果我们把`global_max` 加回来, 那么就是:

```python
def kadane_algorithm(arr):
    dp = [0] * len(arr)
    dp[0] = arr[0]
    global_max = arr[0]

    for i in range(1, len(arr)):
        dp[i] = max(arr[i], dp[i-1] + arr[i])
        global_max = max(global_max, dp[i])

    return global_max
```

这时候最后`global_max` 就是以每一个index结尾的最大的subarray sum中的最大值.

# 总结

了解了这个algo的原理, 以后就不会忘记了. 同时也可以用这个思路去解决其他的问题.

- 比如这道leetcode的[题目](https://leetcode.com/problems/maximum-subarray-sum-with-one-deletion/)
