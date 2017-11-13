---
title: elastalert微信报警
date: 2017-02-16 16:27:53
tags: [elk,elkstasck,ElastAlert]
categories: [elkstasck]
---

针对国人微信使用率较高的情况，开发了三个ElastAlert微信插件(shell,python,java)

<!-- more -->

## 简介
ElastAlert支持以下方式报警

* Command
* Email
* JIRA
* OpsGenie
* SNS
* HipChat
* Slack
* Telegram
* Debug
* Stomp

Email 优点是免费，可追溯(不手动删除情况下),配置方便。缺点是查看不及时(QQ邮箱除外，弹窗提示，我服)，阅读不方便，大部分人都是使用PC阅读邮件

使用Command调用短信接口推送短信，成本高，信息少且单一，不及时（停机时，丢失信息）

详细分析 参见 [为何使用微信企业号团队号][为何使用微信企业号团队号]

## ElastAlert Python 插件

### 准备工作
1. Elasticsearch 5.2.0
2. ElastAlert 0.1.8

### 步骤

具体ElastAlert安装以及使用Email报警，参见我写的另外一篇文章 [ElastAlert 基于Elasticsearch的监控告警](https://anjia.ml/2017/02/14/elasticsearch-elastalert/)

```bash
$ cd ~/

$ git clone https://github.com/Yelp/elastalert.git

$ cd elastalert

$ wget -P ~/elastalert/elastalert_modules/ wget https://raw.githubusercontent.com/anjia0532/elastalert-wechat-plugin/master/elastalert_modules/wechat_qiye_alert.py

$ touch ~/elastalert/elastalert_modules/__init__.py

$ cp  config.yaml.example config.yaml

$ vi example_rules/example_frequency.yaml
```
```yaml
# From example_rules/example_frequency.yaml
#es_host: elasticsearch.example.com
#es_port: 14900
name: Example rule

type: frequency

index: logstash-*

num_events: 1

timeframe:
    minutes: 1

filter:
- term:
    _type: "test"

# (Required)
# The alert is use when a match is found
alert:
- "elastalert_modules.wechat_qiye_alert.WeChatAlerter"

#后台登陆后【设置】->【权限管理】->【普通管理组】->【创建并设置通讯录和应用权限】->【CorpID，Secret】
#设置微信企业号的appid
corp_id: xx
#设置微信企业号的Secret
secret: xx
#后台登陆后【应用中心】->【选择应用】->【应用id】
#设置微信企业号应用id
agent_id: xx
#部门id
party_id: xx
#用户微信号
user_id: xx
# 标签id
tag_id: xx
```


```bash
$ python -m elastalert.elastalert --verbose --rule example_rules/example_frequency.yaml

$ curl -X POST 'http://127.0.0.1:9200/logstash-'$(date +%Y.%m.%d)'/test' -d '{"@timestamp": "'$(date +%Y-%m-%d'T'%T%z)'","field": "value"}'

INFO:elastalert:Starting up
INFO:elastalert:Queried rule Example rule from 2017-02-16 17:16 CST to 2017-02-16 17:25 CST: 1 / 1 hits
{u'errcode': 0, u'errmsg': u'ok'}
INFO:elastalert:发送消息给 xxx
INFO:elastalert:Ran Example rule from 2017-02-16 16:31 CST to 2017-02-16 17:25 CST: 1 query hits, 1 matches, 2 alerts sent
INFO:elastalert:Sleeping for 57 seconds
```

![elastalert-wechat-plugin](https://ooo.0o0.ooo/2017/02/16/58a5712d54ddd.png)

部分代码参考 [python与shell通过微信企业号发送消息][python-shell-wechat]

## ElastAlert Command之java版

### 准备工作
1. [申请企业号][weixin-qiye] 具体自行百度
2. [安装Git][git]
3. [Java 1.8+][jdk]
4. [Maven][maven]

### 步骤

参见我的项目 [anjia0532/weixin-qiye-alert][weixin-qiye-alert]

[python-shell-wechat]: http://www.cnblogs.com/caoguo/p/5668653.html
[为何使用微信企业号团队号]: https://github.com/anjia0532/weixin-qiye-alert#为何使用微信企业号团队号
[weixin-qiye]: https://qy.weixin.qq.com/
[git]: https://git-scm.com/
[jdk]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
[maven]: http://maven.apache.org/download.cgi
[weixin-qiye-alert]: https://github.com/anjia0532/weixin-qiye-alert
