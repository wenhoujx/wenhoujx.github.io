---
layout: post
title:  "Setup a cheap cron server"
date:   2023-10-26 23:51:45 -0400
categories: cloud cron 
---
Situation: I want a text msg notification when tickets become available again. 

Often the website doesn't have this feature, I need a cron running on remote sever that sends me notifications when conditions are met.

## Overall

I need two components 

1. get a cheap server 
   - remote server to make sure it run 24/7.
   - user `crontab` to schedule tasks.  
2. if conditions are met, send text msg via `twilio`. 

## Cron Server

### Google Cloud VM 
There are multiple weays of doing this. (let me know your preferred way)

I choose to get a cheap VM from google-cloud. 

pricing: **ec2-micro** costs $7/month, or $0.23/day

#### Create VM

Here are the steps to create a `ec2-micro` instance. 
  - register a google cloud account, get free starter credit if first time.
  - go to `compute engine`
    - `create new instance`
    - pick `ec2`
    - configure `machine-type`, `shared-core`, pick `ec2-micro` 
  - check the monthly estimate cost. 
  - keep all other defaults. 
  - click `create` to get a `Debian GNU/Linux 11 (bullseye)` machine. 
  
#### Configure crontab 

Once the VM created, you have the option to `ssh` to the box. (google did this well)

click `ssh` should pop up another browser window. 

`crontab` is a standard linux utility, it's a great way to schedule tasks.
Here i only cover the most basics. 

```sh
# list all existing crontab/scheduled-tasks  
crontab -l 
# edit crontab, this will start a vim editor 
crontab -e 
```

**example 1**

Here is an exmaple of example of a cron that happens every 5 minutes

```sh
*/5 * * * * /bin/echo "Your message here"
```

**example 2**

Of course you can put a bash command in a `~/echo.sh` script, and schedule it this way:
```sh
*/5 * * * * ~/echo.sh 
```
Remember to `chmod +x ~/echo.sh` 

**example 3**

Or you can write python or any language. 

here is a `~/post.py`
```python
import requests 
requests.post(...)
```

schedule: 
```sh
*/5 * * * * python3 ~/post.py 
```

## Text Msg Notification 

Use [twilio](https://www.twilio.com/en-us). 

Register an free acount ($15 credit), Get a toll free number from twilio. 

You should get a easy curl example. Try it out and it should send a text `haha` to your phone.   
```sh
curl 'https://api.twilio.com/2010-04-01/Accounts/[account_id]/Messages.json' -X POST \
--data-urlencode 'To=+[YourPhoneNumber]' \
--data-urlencode 'Body=haha' \
-u [account_id]:[AuthToken]
```

### Wire It To Remote Cron Server

You can write the script with either `bash` or `python`, here i give a `python3` example. 

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

```sh
crontab -e
```

```sh 
# edit crontab
*/5 * * * * python3 ~/query_ticket.py >> output.log 2>&1
```




