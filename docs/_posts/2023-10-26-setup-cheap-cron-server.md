---
layout: post
title:  "云端的cron server"
date:   2023-10-26 23:51:45 -0400
categories: cloud cron 
---

**问题**: 我想要短信提醒我网上抢票有票了. 

**解决**: 搞一个云端的cron server, 每5分钟 check 有没有放票, 如果有, 给我发短信.


## 干货

需要两个components

1. 一个便宜的云端的server 
   - 这样可以保证24小时不间断的check 有没有票
   - 在server上用`crontab` schedule tasks
2. 如果有票了, 给我发短信, 这里用到了`twilio`

## Cron Server

### Google Cloud VM 
云端server有很多, 这里用到了google cloud VM. 
- 因为我自己平时用不少google cloud的东西, 所以比较熟悉, aws的也是一样的. 
- 我自己有google cloud的免费credit (新手注册也有)

收费: **ec2-micro** costs $7/month, or $0.23/day

#### 创建一个 VM

创建 `ec2-micro` instance 的步骤 
  - register a google cloud account, get free starter credit if first time.
  - go to `compute engine`
    - `create new instance`
    - pick `ec2`
    - configure `machine-type`, `shared-core`, pick `ec2-micro` 
  - check the monthly estimate cost. 
  - keep all other defaults. 
  - click `create` to get a `Debian GNU/Linux 11 (bullseye)` machine. 
  
#### configure crontab 

一旦VM创建好了, google有一个很好用的`ssh`的功能, 会弹出一个新的window, 而且已经`ssh` 到远端的machine上了.

`crontab` 是linux 的常用的 task schedule的tool. 这里只cover最基本的. 

```sh
# 列出所有的 crontab/scheduled-tasks  
crontab -l 
# edit crontab task, 这里会默认用到vim或者nano
crontab -e 
```

**栗子1**

每5分钟 叫唤一声. 

```sh
*/5 * * * * /bin/echo "Your message here"
```

**栗子2**

当然可以每五分钟 让一个bash script 叫唤一声

```sh
*/5 * * * * ~/echo.sh 
```
记得要 `chmod +x ~/echo.sh`.  

**栗子3**

或者你也可以写python, bash有时候挺难写的. 

here is a `~/post.py`
```python
import requests 
requests.post(...)
```

schedule: 
```sh
*/5 * * * * python3 ~/post.py 
```

## 短信提示

用 [twilio](https://www.twilio.com/en-us). 你也可以用别的.  

注册一个账号 ($15 credit), `twilio` 会给你分配一个免费的toll free 800 号码用来发短信. 

`twilio` 会告诉你一个简单的 `curl` 给自己发一个`haha` 的短信, 如下
```sh
curl 'https://api.twilio.com/2010-04-01/Accounts/[account_id]/Messages.json' -X POST \
--data-urlencode 'To=+[YourPhoneNumber]' \
--data-urlencode 'Body=haha' \
-u [account_id]:[AuthToken]
```

### 全部放一起. 

写一个`bash` 或者 `python` script 检测有没有票, (这里略过). 
如果有, 给我发短信.

```python
# ~/query_ticket.py
import requests 

response = requests.get(...) # get response from the ticket API
has_ticket = reponse.json().get(...).get("remaining_tickets") != 0
if has_ticket: 
    requests.post("'https://api.twilio.com/2010-04-01/Accounts/[account_id]/Messages.json",
        data = {
          "From": ... , 
          "To": ... ,
          "Body": ... ,
        }
    )
```

schedule 
```sh
crontab -e
```

可以加入logging
```sh 
# edit crontab
*/5 * * * * python3 ~/query_ticket.py >> output.log 2>&1
```

这样就好啦, 我已经用这个方法抢到了 shennadoah old-rag的票了, 真是手慢无. 




