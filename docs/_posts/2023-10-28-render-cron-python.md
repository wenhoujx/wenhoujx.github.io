---
layout: post
title:  "用Render Paas 快速部署python的 cron job"
date:   2023-10-28 13:00:00 -0400
categories: cron render python
published: true
---

**问题**:
在之前的 post [抢票用的云端的cron server]({% post_url 2023-10-26-setup-cheap-cron-server %}) 中我们用`google cloud`实现了一个`cron` server, 但是需要注册, 创建VM, 配置, 等等, 比较麻烦.

如何快速部署一个`cron` server, 不需要但是各种`devops`?

**解决**:
`Paas` = `platform as a service` 是一种帮助你快速部署应用的云服务, 不需要自己创建 配置等等的`devops`.

市面上有很多种这种产品, 比如`heroku`, `fly.io`, `render`等等. 这里我们用`render`来实现一个`cron` server, 只需要几行代码, 一分钟就可以搞定.

## 干货

### Render PaaS

platform as a service 让你可以专注于功能的实现, 而不是纠结于各种`devops`的配置.

`render` 这个产品可以自动连接到version controlled github repo, 然后自动部署. 甚至可以自动用`shell` 查看VM的情况.

### 注册render

首先注册一个 免费的 [`render`](https://www.render.com/) 的账号.

### 创建一个 git repo

这里我们把所有的cron 的代码放在一个git repo里面, 然后render会自动部署这个repo.

写一个很简单的 `python` 的脚本.

```python
print("hello world")
```

这是我们的folder structure:

- crons/
  - echo.py

把这个repo push 到github上的同名的 `crons` repo.

具体可见 [github repo crons](https://github.com/wenhoujx/crons)

### 链接render 和 github

在`render`中点击`new+` -> 选择`cron job` -> 选择`build and deploy from github` -> 链接`crons` 这个repo.

- 随便提供一个名字
- 选择 `region`
- `Runtime` 选择 `python`
- `build command` 这里我们没有任何dependencies, 但是是一定要填的. 我们随便填一个 `echo building` 就好.
- `schedule` 这里是`crontab` 的syntax, 我想要每5分钟跑一次, 所以填 `*/5 * * * *`
- `command` 这是最重要的, 我们要调用之前的 `echo.py` 这个文件, 所以填 `python hello.py`
- 选择最便宜的`instance`, 每分钟只要 `0.016`美分, 注意是美分.

创建cron job `create cron job`, 就完成啦!

### 测试

在`cron job` 中可以选择`logs` 看到没五分钟就会`print` 一次 `hello world`.

![render-python-cron](/assets/images/render-cron-python-logs.jpg)

### secrets

很多时候我们的 `python` script 会用到一些 `secrets`, 比如短信通知服务twilio的`account_token`.

如果你的github repo是private, 那你可以放一些secrets, 不过如果是public repo, 你需要把你的secret放在`render`中.

#### 添加secret

在`Render` 中点击`Environment`, 添加自己的secret:

![render-secret](/assets/images/render-secret.jpg)
这里我添加了一个`secret-foo`的secret

添加好了之后去 `shell` 中看看到底是什么样子的, 可以看到`render` 把`secret-foo` file 自动的放在`repo`的root, 而且内容就是我们想要的secret value.

![render-secret-shells](/assets/images/render-secret-shell.jpg)

#### 使用secret

我们可以改动`echo.py` 来使用这个secret.

```python
with open("secret-foo", "r") as f:
    secret_foo = f.read()

print(secret_foo)
```

push到github之后, render会自动deploy(部署)

然后我们去`logs`中看到我们的secret被打印出来了.
![render-secret-logs-2](/assets/images/render-secret-logs-2.jpg)

## 最后

`render`只是继 `heroku` 被`salesforce` 阉割之后新生的一个比较好用的`PaaS`产品. 以后还会介绍 `fly.io` 和其他便宜又好用, 适合独立开发者的产品. 甚至会聊到 `kubernetes`.
