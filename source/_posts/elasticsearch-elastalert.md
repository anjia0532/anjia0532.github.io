---
title: ElastAlert 基于Elasticsearch的监控告警
date: 2017-02-14 08:27:47
tags: [elk,elkstasck,ElastAlert]
categories: [elkstasck]
---

Elastalert是Yelp公司用python2写的一个报警框架(目前支持python2.6和2.7，不支持3.x),github地址为 [https://github.com/Yelp/elastalert](https://github.com/Yelp/elastalert)

<!-- more -->

## 环境

Ubuntu 16.10(内核 4.8.0-37-generic)

elasticsearch 5.2.0

logstash 5.2.0

kibana 5.2.0

## 依赖

参见 [http://elastalert.readthedocs.io/en/latest/running_elastalert.html#requirements](http://elastalert.readthedocs.io/en/latest/running_elastalert.html#requirements)

- Elasticsearch
- ISO8601 or Unix timestamped data
- Python 2.6 or 2.7
- pip, see requirements.txt


## 安装Elastalert

安装之前先运行 `python --version`查看python的版本

```bash
python --version

Python 2.7.12+

#如果2.6或者2.7则正常，如果是3.x则需要改成python2.x
#假设本机装了python 2和3 可以将/usr/bin/python的软连接指向 python2
```

```bash
# 下载最新elastalert
git clone https://github.com/Yelp/elastalert.git

# 安装模块
sudo python setup.py install

sudo pip install -r requirements.txt

```

安装完后，会在 /usr/local/bin/ 下生成4个elastalert命令
```bash

$ ll /usr/local/bin/elastalert*
-rwxr-xr-x 1 root root 396 2月  14 10:03 /usr/local/bin/elastalert
-rwxr-xr-x 1 root root 422 2月  14 10:03 /usr/local/bin/elastalert-create-index
-rwxr-xr-x 1 root root 430 2月  14 10:03 /usr/local/bin/elastalert-rule-from-kibana
-rwxr-xr-x 1 root root 416 2月  14 10:03 /usr/local/bin/elastalert-test-rule

```

## 设置elasticsearch索引

参见 [setting-up-elasticsearch][setting-up-elasticsearch] 

`elastalert-create-index` 这个命令会在elasticsearch创建索引，这不是必须的步骤，但是强烈建议创建。因为对于，审计，测试很有用，并且重启elastalert不影响计数和发送`alert`,默认情况下，创建的索引叫 `elastalert_status`

```bash
$ elastalert-create-index
New index name (Default elastalert_status)
Name of existing index to copy (Default None)
New index elastalert_status created
Done!
```

具体生成的数据，请参见 [ElastAlert Metadata Index][metadata]

## 设置配置文件和规则Rule

```bash
cp ~/elastalert/config.yaml.example ~/elastalert/config.yaml
vi ~/elastalert/config.yaml
```

```yaml
# This is the folder that contains the rule yaml files
# Any .yaml file will be loaded as a rule
rules_folder: example_rules

# How often ElastAlert will query Elasticsearch
# The unit can be anything from weeks to seconds
run_every:
  minutes: 1

# ElastAlert will buffer results from the most recent
# period of time, in case some log sources are not in real time
buffer_time:
  minutes: 15

# The Elasticsearch hostname for metadata writeback
# Note that every rule can have its own Elasticsearch host
es_host: 127.0.0.1

# The Elasticsearch port
es_port: 9200

# Optional URL prefix for Elasticsearch
#es_url_prefix: elasticsearch

# Connect with TLS to Elasticsearch
#use_ssl: True

# Verify TLS certificates
#verify_certs: True

# GET request with body is the default option for Elasticsearch.
# If it fails for some reason, you can pass 'GET', 'POST' or 'source'.
# See http://elasticsearch-py.readthedocs.io/en/master/connection.html?highlight=send_get_body_as#transport
# for details
#es_send_get_body_as: GET

# Option basic-auth username and password for Elasticsearch
#es_username: someusername
#es_password: somepassword

# The index on es_host which is used for metadata storage
# This can be a unmapped index, but it is recommended that you run
# elastalert-create-index to set a mapping
writeback_index: elastalert_status

# If an alert fails for some reason, ElastAlert will retry
# sending the alert until this time period has elapsed
alert_time_limit:
  days: 1

```

```bash
#注意将${userName}替换成具体用户名
vi /home/${userName}/elastalert/example_rules/smtp_auth_file.yaml
```
```yaml
#发送邮件的邮箱
user: xxx@163.com
#不是邮箱密码，是设置的POP3密码
password: xxx
```
```bash
vi ~/elastalert/example_rules/example_frequency.yaml
```
参见 [creating-a-rule][creating-a-rule]
```yaml
# From example_rules/example_frequency.yaml
#es_host: elasticsearch.example.com
#es_port: 14900
name: Example rule
type: frequency
index: logstash-*
#限定时间内，发生事件次数
num_events: 1
#限定时间刻度
timeframe:
    #1分钟
    minutes: 1

filter:
- query:
    query_string:
      query: "field: value"

#SMTP协议的邮件服务器相关配置
#smtp.163.com是网易163邮箱的smtp服务器
#登陆163邮箱后，找到 【设置】>【POP3/SMTP/IMAP】>开启，然后设置【客户端授权密码】
smtp_host: smtp.163.com
smtp_port: 25

#用户认证文件，需要user和password两个属性
#注意将${userName}替换成具体用户名
smtp_auth_file: /home/${userName}/elastalert/example_rules/smtp_auth_file.yaml
#回复给那个邮箱
email_reply_to: xxx@163.com
#从哪个邮箱发送
from_addr: xxx@163.com

# (Required)
# The alert is use when a match is found
alert:
- "email"

# (required, email specific)
# a list of email addresses to send alerts to
email:
#接收报警邮件的邮箱
- "xxxx@qq.com"
```

## 测试规则

参见 [Testing Your Rule][testing-your-rule]

```bash
elastalert-test-rule ~/elastalert/example_rules/example_frequency.yaml
```

具体配置，参见 [commonconfig][commonconfig]

## 运行
```bash
$ cd ~/elastalert
$ python -m elastalert.elastalert --verbose --rule example_frequency.yaml

INFO:elastalert:Starting up
INFO:elastalert:Queried rule Example rule from 2017-02-14 11:08 CST to 2017-02-14 11:09 CST: 0 / 0 hits
INFO:elastalert:Ran Example rule from 2017-02-14 11:08 CST to 2017-02-14 11:09 CST: 0 query hits, 0 matches, 0 alerts sent
INFO:elastalert:Sleeping for 59 seconds

```
```bash
$ curl -X POST "http://127.0.0.1:9200/logstash-2017.02.14/test"  -d '{
"@timestamp": "2017-02-14T03:10:46.000Z",
"field": "value"
}'

# 返回 {"_index":"logstash-2017.02.14","_type":"test","_id":"AVo6oVCnFreCcJPhQqgX","_version":1,"result":"created","shards":{"total":2,"successful":1,"failed":0},"created":true}

```
**@timestamp的时间是UTC时间，换算方式北京时间（东八区）减8小时，例如2017-02-14 11:21:50的UTC时间是 2017-02-14 03:21:50**
```bash
#如果正常，会输出如下信息

INFO:elastalert:Queried rule Example rule from 2017-02-14 11:08 CST to 2017-02-14 11:19 CST: 2 / 2 hits
INFO:elastalert:Alert for Example rule at 2017-02-14T03:10:46Z:
INFO:elastalert:Example rule

At least 1 events occurred between 2017-02-14 11:09 CST and 2017-02-14 11:10 CST

@timestamp: 2017-02-14T03:10:46Z
_id: AVo6oVCnFreCcJPhQqgX
_index: logstash-2017.02.14
_type: test
field: value
num_hits: 2
num_matches: 1

INFO:elastalert:Sent email to ['xxx@qq.com']
INFO:elastalert:Ran Example rule from 2017-02-14 11:08 CST to 2017-02-14 11:19 CST: 2 query hits, 1 matches, 2 alerts sent
INFO:elastalert:Sleeping for 59 seconds

```

## Alert

![成功报警](https://ooo.0o0.ooo/2017/02/14/58a27e882df14.png)

[setting-up-elasticsearch]: http://elastalert.readthedocs.io/en/latest/running_elastalert.html#setting-up-elasticsearch
[metadata]: http://elastalert.readthedocs.io/en/latest/elastalert_status.html#metadata
[creating-a-rule]: http://elastalert.readthedocs.io/en/latest/running_elastalert.html#creating-a-rule
[commonconfig]: http://elastalert.readthedocs.io/en/latest/ruletypes.html#commonconfig
[testing-your-rule]: http://elastalert.readthedocs.io/en/latest/running_elastalert.html#testing-your-rule
