---
title: elasticsearch按照日期定时删除索引
date: 2017-04-06 14:10:47
tags: [elk,elkstasck,curator]
categories: [elkstasck]
---

使用elkstack作为日志分析工具，采集nginx访问日志，项目log日志，心跳检测日志，服务器度量日志等，每天产生大量索引(Index)，占用磁盘空间。对于过期数据需要进行删除来释放磁盘空间。

<!-- more -->

### 使用官网_delete_by_query进行删除
[官网文档--Delete By Query API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-delete-by-query.html)

```bash
curl -u 用户名:密码  -H'Content-Type:application/json' -d'{
    "query": {
        "range": {
            "@timestamp": {
                "lt": "now-7d",
                "format": "epoch_millis"
            }
        }
    }
}
' -XPOST "http://127.0.0.1:9200/*-*/_delete_by_query?pretty"
```

**解释**

`-u`是格式为`userName:password`，使用`Basic Auth`进行登录。如果`elasticsearch`没有使用类似`x-pack`进行安全登录，则不需要加`-u参数`

`-H`是指定文档类型是json格式

`-XPOST`是指定用`POST`方式请求

`-d`是指定`body`内容

```javascript
{
    "query": {
        "range": { //范围
            "@timestamp": {//时间字段
                "lt": "now-7d",//lt是小于(<)，lte是小于等于(<=),gt是大于(>),gte是大于等于(>=),now-7d是当前时间减7天
                "format": "epoch_millis"
            }
        }
    }
}
```

定时删除
```bash
$ crontab -e

* 0 * * * /usr/bin/curl -u username:password  -H'Content-Type:application/json' -d'{"query":{"range":{"@timestamp":{"lt":"now-7d","format":"epoch_millis"}}}}' -XPOST "http://127.0.0.1:9200/*-*/_delete_by_query?pretty" > /tmp/elk_clean.txt
```

每天0点删除超过7天的无效索引

优点：

- 不依赖第三方插件或者代码

- 简单易理解

- 不需要指定索引名称可用`*`通配符删除

缺点：
- 效率低

### 使用sh脚本删除
在stackoverflow看到一个帖子 [Removing old indices in elasticsearch#answer-39746705](http://stackoverflow.com/questions/33430055/removing-old-indices-in-elasticsearch#answer-39746705)
```bash
#!/bin/bash
searchIndex=logstash-monitor
elastic_url=logging.core.k94.kvk.nl
elastic_port=9200

date2stamp () {
    date --utc --date "$1" +%s
}

dateDiff (){
    case $1 in
        -s)   sec=1;      shift;;
        -m)   sec=60;     shift;;
        -h)   sec=3600;   shift;;
        -d)   sec=86400;  shift;;
        *)    sec=86400;;
    esac
    dte1=$(date2stamp $1)
    dte2=$(date2stamp $2)
    diffSec=$((dte2-dte1))
    if ((diffSec < 0)); then abs=-1; else abs=1; fi
    echo $((diffSec/sec*abs))
}

for index in $(curl -s "${elastic_url}:${elastic_port}/_cat/indices?v" |     grep -E " ${searchIndex}-20[0-9][0-9]\.[0-1][0-9]\.[0-3][0-9]" | awk '{     print $3 }'); do
  date=$(echo ${index: -10} | sed 's/\./-/g')
  cond=$(date +%Y-%m-%d)
  diff=$(dateDiff -d $date $cond)
  echo -n "${index} (${diff})"
  if [ $diff -gt 1 ]; then
    echo " / DELETE"
    # curl -XDELETE "${elastic_url}:${elastic_port}/${index}?pretty"
  else
    echo ""
  fi
done    
```
使用了 `_cat/indices`api。

### 使用 curator

支持windows[zip](https://www.elastic.co/guide/en/elasticsearch/client/curator/5.0/windows-zip.html),[msi](https://www.elastic.co/guide/en/elasticsearch/client/curator/5.0/windows-msi.html),和linux[apt](https://www.elastic.co/guide/en/elasticsearch/client/curator/5.0/apt-repository.html),[yum](https://www.elastic.co/guide/en/elasticsearch/client/curator/5.0/yum-repository.html)

[Curator Reference](https://www.elastic.co/guide/en/elasticsearch/client/curator/5.0/index.html) [github-curator](https://github.com/elastic/curator)

#### 安装
[安装](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/installation.html)

#### 配置

参考 http://stackoverflow.com/questions/33430055/removing-old-indices-in-elasticsearch#answer-42268400

1.config文件

```yaml
---
# Remember, leave a key empty if there is no value.  None will be a string,
# not a Python "NoneType"
client:
  hosts:
    * 127.0.0.1
  port: 9200
  url_prefix:
  use_ssl: False
  certificate:
  client_cert:
  client_key:
  ssl_no_validate: False
  http_auth: username:password
  timeout:
  master_only: True

logging:
  loglevel: INFO
  logfile:
  logformat: default
  #blacklist: ['elasticsearch', 'urllib3']
```

2.action文件

```yaml
---
actions:
  1:
    action: delete_indices
    description: >-
      Delete indices older than 7 days (based on index name), for logstash-
      prefixed indices. Ignore the error if the filter does not result in an
      actionable list of indices (ignore_empty_list) and exit cleanly.
    options:
      ignore_empty_list: True
      timeout_override:
      continue_if_exception: False
      disable_action: False
    filters:
    * filtertype: pattern
      kind: prefix
      value: logstash-
      exclude:
    * filtertype: age
      source: name
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 7
      exclude:
```

这里是用`index-'%Y.%m.%d'`进行匹配，如果是按照索引创建日期来删除，`source: creation_date` 参见 https://www.elastic.co/guide/en/elasticsearch/client/curator/current/fe_source.html#_creation_date

3.运行

```
curator --config /path/config_file.yml /path/action_file.yml
```

别忘了加定时任务`crontab -e`