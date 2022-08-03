---
title: 071-基于flattened解决apisix写入es导致的字段爆炸问题
urlname: es-flattend
date: '2022-03-01 19:35:21 +0800'
tags:
  - es
  - elasticsearch
  - elk
  - efk
  - apisix
categories:
  - elk
  - apisix
---

> 这是坚持技术写作计划（含翻译）的第 71 篇，定个小目标 999，每周最少 2 篇。

[Apisix](https://apisix.apache.org/) 支持日志输出到 [kafka](https://apisix.apache.org/zh/docs/apisix/plugins/kafka-logger), [tcp](https://apisix.apache.org/zh/docs/apisix/plugins/tcp-logger), [udb](https://apisix.apache.org/zh/docs/apisix/plugins/udp-logger), [syslog](https://apisix.apache.org/zh/docs/apisix/plugins/syslog) 等多种方式

主流的不管是 ELK 还是 EFK, 都是用 Elasticsearch 是用于存储数据，Logstash/Fluentd/Filebeat 是用于摄取数据，Kibana 是用于展示数据(Grafana 也支持 ES 作为数据源)

其中：

- [Logstash](https://www.elastic.co/cn/logstash/) 支持大部分, 即所谓的 ELK，参见 [Logstash Input Plugins](https://www.elastic.co/guide/en/logstash/current/input-plugins.html) ,
- [Fluentd](https://www.fluentd.org/), 即所谓的 EFK 可以参考 [Fluentd Input Plugins](https://docs.fluentd.org/input)
- [Filebeat](https://www.elastic.co/cn/beats/filebeat), 即所谓的 ELK 可以参考 [Filebeat Input Types](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-filebeat-options.html#filebeat-input-types)

还有一些逐渐流行的方案，用其他存储方案替代 ES ,一般来说，普遍比 ES 好运维，如果没有倒排索引的需求，及历史包袱的情况下，建议别用 ES 的方案

- 比如 [Grafana Loki](https://grafana.com/oss/loki/) 作为存储源，配合自家的 Grafana 作为展示
- 使用 [Clickhouse ](https://clickhouse.com/docs/zh/)作为存储源, 参考 [快手、携程等公司转战到 ClickHouse，ES 难道不行了？](https://www.jianshu.com/p/26c05246ecfc)，[干货 | 携程 ClickHouse 日志分析实践](https://cloud.tencent.com/developer/article/1585886)
- 使用 [TDengine](https://www.taosdata.com/?zh), 参考 [TDengine 官方文档-高效写入数据](https://www.taosdata.com/docs/cn/v2.0/insert)
- ...

本文主要介绍 Elasticsearch 默认 Mapping 如果不做特殊设置，默认为 `dynamic`，导致 Apisix 输入到 ES 的日志，不停动态添加字段,直到超过 `index.mapping.total_fields.limit`而报错(有新增字段的插入不了)，通俗来说就是字段爆炸, 主要是 `request.headers`,`request.querystring`两个对象。

<!-- more -->

## 问题描述

### 查看当前索引字段上限

如果没改过，则默认是 `1000` ，参考文档 [Mapping limit settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-settings-limit.html#mapping-settings-limit)
如果不确定，可以通过 API 查询，查看返回值有没有 `index.mapping.total_fields.limit`

```
# 查询
GET apisix-*/_settings

# 修改 index.mapping.total_fields.limit 为 2000
PUT apisix-*/_settings
{
  "index.mapping.total_fields.limit": 2000
}
```

### 查看当前索引字段数

打开 Kibana `Stack Management > 索引模式 > apisix-*` 查看字段数量，如果超过 300，最好是干预下，否则字段爆炸是早晚的事。而且需要注意的是，这个问题不是很明显，如果超过字段上限后不新增字段，日志能正常插入，只有新增字段才会报错，也就是说，会间歇性丢日志。

## 解决方案

### ES 7.3 之前的方案

- 通过 `dynamic`改为 `false` (缺点，新增字段不能被索引)
- 通过 `dynamic` 设置为 `strict`(缺点，新增字段直接报错)

参考文档 [Dynamic field mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html)

### ES 7.3 及之后的方案

无脑使用 [flattened](https://www.elastic.co/guide/en/elasticsearch/reference/current/flattened.html)

打开 kibana 的 DevTools 控制台执行下面脚本，假设你的 Apisix 的日志索引是 `apisix-*`，注意索引模板是要在索引生成之前设置才有效，如果是想修改当前索引，则使用 `PUT apisix-*/_settings` API

```
PUT _index_template/apisix_template
{
  # 匹配索引
  "index_patterns": ["apisix-*"],

  # 模板
  "template": {

    # 设置，此处可忽略，可以只改映射 mappings 部分
    "settings": {
      # 每30秒刷新一次
      "refresh_interval": "30s",
      # 3分片
      "number_of_shards": "3",
      # 1副本
      "number_of_replicas": "1",
    },

    # 映射
    "mappings": {
      "properties": {
        # 将 request.headers,request.querystring,response.headers 设置为 flattened 类型
        "request.headers": {
          "type": "flattened"
        },
        "request.querystring": {
          "type": "flattened"
        },
        "response.headers": {
          "type": "flattened"
        }
      }
    }
  }
}
```

引用 [Elasticsearch 字段膨胀不要怕，Flattened 类型解千愁！](https://mp.weixin.qq.com/s/jVs-jOSPXZ07ikcZ0d0SqQ) 介绍的 flattened 缺点

> 每当面临 Flattened 扁平化对象的决定时，在选型 Elasticsearch 扁平化数据类型时，我们需要考虑以下几个关键限制：
> Flattened 类型支持的查询类型目前仅限于以下几种：
>
> - term
> - terms
> - terms_set
> - prefix
> - range
> - match and multi_match
> - query_string and simple_query_string
> - exists
>
> Flattened 不支持的查询类型如下：
>
> - 无法执行涉及数字计算的查询，例如：range query。
> - 无法支持高亮查询。
> - 尽管支持诸如 term 聚合之类的聚合，但不支持处理诸如“histograms”或“date_histograms”之类的数值数据的聚合。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/e78fa84f96fef4e733J60tq8EA~~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/03/01/es-flattend)
- [我的掘金](https://juejin.cn/post/7070054075504525325/)
- [Elasticsearch 字段膨胀不要怕，Flattened 类型解千愁！](https://mp.weixin.qq.com/s/jVs-jOSPXZ07ikcZ0d0SqQ)
