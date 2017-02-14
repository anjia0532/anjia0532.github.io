---
title: ElastAlert--����Elasticsearch�ļ�ظ澯
date: 2017-02-14 08:27:47
tags: [elk,elkstasck,ElastAlert]
categories: [elkstasck]
---

Elastalert��Yelp��˾��python2д��һ���������(Ŀǰ֧��python2.6��2.7����֧��3.x),github��ַΪ [https://github.com/Yelp/elastalert](https://github.com/Yelp/elastalert)

<!-- more -->

## ����

Ubuntu 16.10(�ں� 4.8.0-37-generic)

elasticsearch 5.2.0

logstash 5.2.0

kibana 5.2.0

## ����

�μ� [http://elastalert.readthedocs.io/en/latest/running_elastalert.html#requirements](http://elastalert.readthedocs.io/en/latest/running_elastalert.html#requirements)

- Elasticsearch
- ISO8601 or Unix timestamped data
- Python 2.6 or 2.7
- pip, see requirements.txt


## ��װElastalert

��װ֮ǰ������ `python --version`�鿴python�İ汾

```bash
python --version

Python 2.7.12+

#���2.6����2.7�������������3.x����Ҫ�ĳ�python2.x
#���豾��װ��python 2��3 ���Խ�/usr/bin/python��������ָ�� python2
```

```bash
# ��������elastalert
git clone https://github.com/Yelp/elastalert.git

# ��װģ��
sudo python setup.py install

sudo pip install -r requirements.txt

```

��װ��󣬻��� /usr/local/bin/ ������4��elastalert����
```bash

$ ll /usr/local/bin/elastalert*
-rwxr-xr-x 1 root root 396 2��  14 10:03 /usr/local/bin/elastalert
-rwxr-xr-x 1 root root 422 2��  14 10:03 /usr/local/bin/elastalert-create-index
-rwxr-xr-x 1 root root 430 2��  14 10:03 /usr/local/bin/elastalert-rule-from-kibana
-rwxr-xr-x 1 root root 416 2��  14 10:03 /usr/local/bin/elastalert-test-rule

```

## ����elasticsearch����

�μ� [setting-up-elasticsearch][setting-up-elasticsearch] 

`elastalert-create-index` ����������elasticsearch�����������ⲻ�Ǳ���Ĳ��裬����ǿ�ҽ��鴴������Ϊ���ڣ���ƣ����Ժ����ã���������elastalert��Ӱ������ͷ���`alert`,Ĭ������£������������� `elastalert_status`

```bash
$ elastalert-create-index
New index name (Default elastalert_status)
Name of existing index to copy (Default None)
New index elastalert_status created
Done!
```

�������ɵ����ݣ���μ� [ElastAlert Metadata Index][metadata]

## ���������ļ��͹���Rule

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
#ע�⽫${userName}�滻�ɾ����û���
vi /home/${userName}/elastalert/example_rules/smtp_auth_file.yaml
```
```yaml
#�����ʼ�������
user: xxx@163.com
#�����������룬�����õ�POP3����
password: xxx
```
```bash
vi ~/elastalert/example_rules/example_frequency.yaml
```
�μ� [creating-a-rule][creating-a-rule]
```yaml
# From example_rules/example_frequency.yaml
#es_host: elasticsearch.example.com
#es_port: 14900
name: Example rule
type: frequency
index: logstash-*
#�޶�ʱ���ڣ������¼�����
num_events: 1
#�޶�ʱ��̶�
timeframe:
    #1����
    minutes: 1

filter:
- query:
    query_string:
      query: "field: value"

#SMTPЭ����ʼ��������������
#smtp.163.com������163�����smtp������
#��½163������ҵ� �����á�>��POP3/SMTP/IMAP��>������Ȼ�����á��ͻ�����Ȩ���롿
smtp_host: smtp.163.com
smtp_port: 25

#�û���֤�ļ�����Ҫuser��password��������
#ע�⽫${userName}�滻�ɾ����û���
smtp_auth_file: /home/${userName}/elastalert/example_rules/smtp_auth_file.yaml
#�ظ����Ǹ�����
email_reply_to: xxx@163.com
#���ĸ����䷢��
from_addr: xxx@163.com

# (Required)
# The alert is use when a match is found
alert:
- "email"

# (required, email specific)
# a list of email addresses to send alerts to
email:
#���ձ����ʼ�������
- "xxxx@qq.com"
```

## ���Թ���

�μ� [Testing Your Rule][testing-your-rule]

```bash
elastalert-test-rule ~/elastalert/example_rules/example_frequency.yaml
```

�������ã��μ� [commonconfig][commonconfig]

## ����
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

# ���� {"_index":"logstash-2017.02.14","_type":"test","_id":"AVo6oVCnFreCcJPhQqgX","_version":1,"result":"created","shards":{"total":2,"successful":1,"failed":0},"created":true}

```
**@timestamp��ʱ����UTCʱ�䣬���㷽ʽ����ʱ�䣨����������8Сʱ������2017-02-14 11:21:50��UTCʱ���� 2017-02-14 03:21:50**
```bash
��������������������Ϣ

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

![�ɹ�����](https://ooo.0o0.ooo/2017/02/14/58a27e882df14.png)

[setting-up-elasticsearch]: http://elastalert.readthedocs.io/en/latest/running_elastalert.html#setting-up-elasticsearch
[metadata]: http://elastalert.readthedocs.io/en/latest/elastalert_status.html#metadata
[creating-a-rule]: http://elastalert.readthedocs.io/en/latest/running_elastalert.html#creating-a-rule
[commonconfig]: http://elastalert.readthedocs.io/en/latest/ruletypes.html#commonconfig
[testing-your-rule]: http://elastalert.readthedocs.io/en/latest/running_elastalert.html#testing-your-rule
