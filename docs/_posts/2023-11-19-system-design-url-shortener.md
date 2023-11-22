---
layout: post
title:  "系统设计: url shortener"
date:   2023-11-19 18:00:00 -0400
categories: system-design google-map routing shortest-path 
published: false
---

**问题**:

如何系统设计 url shortener.

## 需求

基本需求就是给一个 long url, 变成short url, 更美观. 用户可以通过short url 访问到long url.

这个short url都是在我们控制的domain下的, 这样我们才能让browser redirect.

比如一个很长的url:

`https://www.google.com/search?q=how+to+design+url+shortener&oq=how+to+design+url+shortener&aqs=chrome..69i57j0l7.10159j0j7&sourceid=chrome&ie=UTF-8`,

可能他的shorturl是 `https://www.mydomain.com/shorturl`.

注意这里的domain, 这样用户访问的时候会hit我们的server, 我们在用301或者302 的 http response redirect 到long url.

- 301 - permanent redirect
- 302 - temporary redirect

browser 会cache 302 和 301, 这样下次访问的时候就不会再hit我们的server了. (但是我们的server就没有UA统计了)

## 生成short url

有两种方法:

1. 用hash function, 比如md5, sha256, murmurhash, etc.
2. 用
