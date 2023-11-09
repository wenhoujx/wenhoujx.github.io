---
layout: post
title:  "简单好写的快速质数prime test"
date:   2023-11-08 19:36:00 -0400
categories: chatgpt prime-number python
published: true
---

**问题**:

今天刷leetcode的时候需要一个 prime tester, 就随手问一下 `chargpt`, 结果答案惊艳到我了.

```python
def is_prime(n):
    if n <= 1:
        return False
    elif n <= 3:
        return True
    elif n % 2 == 0 or n % 3 == 0:
        return False
    # 到这里还平平无奇. 

    i = 5
    while i * i <= n:
        if n % i == 0 or n % (i + 2) == 0:
            return False
        i += 6 # 这个 +6 是关键

    return True
```

为什么要test `i+2` 还有为什么要 `i += 6` 呢?

## 解答

这个algo的名称是 `6k+-1` algo, 也叫 `6k+1` algo.

用到的数学知识是: 大于3的 `prime number` 一定是 `6k+/-1` 即 `6k-1` 或者 `6k+1`的形式.

这样就解释了为什么要 `+6`, 和为什么要测试 `i+2` ,因为 `i`是从 5开始的, 每次 `+6` 其实 `i` 都满足 `6k-1`, `+2`之后就变成了 `6k+1`.

证明如下:
任何一个大于3的数, 被6除, 余数是 0, 1, 2, 3, 4, 5.

如果余数是 1 或者 5 那就满足这个定理.

下面我们证明余数如果是0, 2, 3, 4, 那么这个数一定不是prime.

- 如果是0, 那么这个数可以被6整除
- 如果是 2,4, 那么这个数可以被2整除
- 如果是3, 那么这个数可以被3整除.

所以任何一个prime number % 6 必然是1 或者 5.

QED.

Chatgpt 统一世界! 🌎
