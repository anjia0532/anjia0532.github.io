---
title: elasticsearch冷热数据分离并使用oss备份老数据
date: 2018-07-30 16:31:40
tags: [elasticsearch,es,hot-warm,oss,backup,curator]
---



从某东看希捷2.5寸企业级硬盘10K转600G在1.6K左右，英特尔ssd 1T 在2-3K左右，（普通民用的便宜，普通硬盘8T 10T 12T 的也不超过2.5K）

怎么在保证扩容方便（不用走冗长的采购审批流程），保证数据安全（oss 持久性SLA是99.999999999% ），前提下，节省存储成本,经过一番研究，发现用oss来做虽然猥琐了点，但是成本最低（目前阿里云oss有活动1T三年99,10T三年999，到期后，就别续费了，新买资源包就行了，数据不用迁移）。

下面讲一下具体步骤，以及遇到的坑。



## 简要介绍

1.  环境 vpc 内网 ，ecs，数据磁盘，oss，elk
2.  热数据存在ecs数据磁盘，冷数据存在oss上，归档数据存在oss上
   1. 热数据(hot)：(根据情况1d-7d,一般日志场景超过1d就不会有高频访问了)，按需设置1-3副本
   2. 冷数据(warm)：(根据情况[2~7]-30天的)，副本0，分片1，应对偶尔的查询需求
   3. 快照(snapshot)：超过30天的，进行归档。

准备[ossfs > 快速安装](https://help.aliyun.com/document_detail/32196.html?spm=a2c4g.11174283.6.1104.IDaiUN) 将oss挂载成warm节点的磁盘

```
$ mkdir /path/to/warm/dir
$ ossfs <bucketname> /path/to/warm/dir -ourl=http://vpc100-oss-cn-<节点名>.aliyuncs.com  -opasswd_file=/path/to/oss-passwds/file -o allow_other -omultireq_max=100 -omultipart_size=20 -oretries=4 -oconnect_timeout=600 -oreadwrite_timeout=600 -omax_stat_cache_size=5000
```



## 设置冷热节点

详见 [“Hot-Warm” Architecture in Elasticsearch 5.x](https://www.elastic.co/blog/hot-warm-architecture-in-elasticsearch-5-x) 中文(像是机译) [Elasticsearch 5.x 版本中的冷热节点架构](https://github.com/ximply/ElasticStackArticle/blob/master/hot-warm-architecture-in-elasticsearch-5-x-in-chinese.md)

> 请别纠结为嘛hot-warm不叫成热-温，而是叫冷热，纯粹是叫着顺口，that's all

#### 修改elasticsearch.yml

1. 在热数据节点的elasticsearch.yml中增加 `node.attr.box_type: hot `,`path.data:/path/to/data` 如果是多块磁盘建议写多个文件夹，加快写入速度，防止磁盘io阻塞 详见 [`path.data` and `path.logs`](https://www.elastic.co/guide/en/elasticsearch/reference/current/path-settings.html)
2. 在冷数据节点的elasticsearch.yml中增加 `node.attr.box_type: warm`,`path.data:/path/to/oss_data`

Note: 此处的`hot`和`warm`是可以随便写的

#### 设置索引模板

```
PUT _template/hot_warm_template
{
    "order": 0,
    "version": 60001,
    "index_patterns": [
      "<indexname>*"
    ],
    "settings" : {
      "index.routing.allocation.require.box_type": "hot"
    ...
}
```

以后新增的索引，都会发往`hot`节点， 需要将上面的template中的`<indexname>`替换成真正的indexname，如果要弄成全部，则可以粗暴的改成`*`



#### 将老数据迁移到warm节点（oss）

**坑1**  开始想着标准存储的dd测速，读100m/s 写40m/s速度足够了，然后热数据直接写入到oss，返回数据会有丢失情况（正常30G的数据，oss只有1-2G），原因不明，改用hot-warm后，因为不会有小文件(forcemerge成大文件了)，转存到oss上不会丢失数据



简单的，可以通过 

```
PUT /indexname/_settings
{ 
  "settings": { 
    "index.routing.allocation.require.box_type": "warm"
  } 
}
```

但是不建议这么做，一个是累（每次都得写命令干?），再就是还得合并索引段(index segments),另外还得减少副本数。

所以，Curator 了解一下？详见[Features](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/about-features.html) 

如果嫌弃安装麻烦，我封装了一个[docker-curator](https://github.com/anjia0532/docker-curator)镜像



##### action_file.yml

```yaml
actions:
  1:
    action: replicas
    description: >-
      超过3天的logstash-%Y.%m.%d副本数设置成0
    options:
      count: 0
      wait_for_completion: True
      disable_action: True
    filters:
    - filtertype: pattern
      kind: prefix
      value: logstash-
    - filtertype: age
      source: creation_date
      direction: older
      unit: days
      unit_count: 3
      
  2:
    action: forcemerge
    description: >-
      强制合并超过3天的logstash-%Y.%m.%d为每片（每个shard不建议超过32G）2个segments(单个segments不建议超过5G)，自动跳过低于2个segments的，防止重复合并
    options:
      max_num_segments: 2
      delay: 120
      timeout_override:
      continue_if_exception: False
      disable_action: True
    filters:
    - filtertype: pattern
      kind: prefix
      value: logstash-
      exclude:
    - filtertype: age
      source: creation_date
      direction: older
      unit: days
      unit_count: 3
      exclude:
    - filtertype: forcemerged
      max_num_segments: 2
      exclude:
  3:
    action: allocation
    description: "将超过3天的logstash-%Y.%m.%d移动到warm节点上"
    options:
      key: box_type
      value: warm
      allocation_type: require
      wait_for_completion: true
      timeout_override:
      continue_if_exception: false
      disable_action: false
    filters:
    - filtertype: pattern
      kind: prefix
      value: logstash-
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 3
```

可以通过`GET _cat/segments` 查看segments count和所属节点

因为warm节点的data指定的是oss挂载的磁盘，所以数据此时已经迁移到oss上了

#### 数据快照备份到oss

如果你想说为啥warm已经放到oss了，还要整快照(备份)

1. 可以不同集群恢复数据
2. 减少es堆内存
3. 关闭用不到的index等
4. 如果是按量付费的oss，归档类型的比标准类型的便宜很多(0.033元/GB/月)

以上

##### 设置共享文件存储

我采用的还是挂载磁盘的方案，不过社区内有大牛基于aliyun-oss-sdk写了个es的存储插件，[elasticsearch-repository-oss](https://github.com/zhichen/elasticsearch-repository-oss/wiki) 但是这个是5.5.3的，如果其他版本，`需要修改plguins/plugin-descriptor.properties中的elasticsearch.version和version,改为自己es集群的版本` 也有别人改好的，参见  [gist#aramalipoor/.env](aramalipoor/.env) ，当然也可以fork后简单的改改，比如升级一下sdk版本，优化一下代码啥的



此处主要讲 挂载磁盘当共享存储用的方案，参见 [shared file system repository](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html#_shared_file_system_repository)
1.挂载oss磁盘为 `/es-data/backup`

2.修改所有master节点和data节点的`elasticsearch.yml`
```yaml
path.repo: ["/es-data/backup"]
```
3.创建共享文件库
```
PUT /_snapshot/my_backup
{
    "type": "fs",
    "settings": {
        "location": "/es-data/backup",
        "compress": true
    }
}
```

action_file.yml

```yaml
actions:
  1:
    action: snapshot
    description: >-
      将超过15天的logstash-%Y.%m.%d 索引按照指定的名字(默认curator-%Y%m%d%H%M%S)备份快照。
      等待快照完成，不跳过存储库文件系统检查，使用别的选项创建快照
    options:
      repository:
      # 如果为空，则默认快照名为 'curator-%Y%m%d%H%M%S'
      name:
      ignore_unavailable: False
      include_global_state: True
      partial: False
      wait_for_completion: True
      skip_repo_fs_check: False
      disable_action: True
    filters:
    - filtertype: pattern
      kind: prefix
      value: logstash-
    - filtertype: age
      source: creation_date
      direction: older
      unit: days
      unit_count: 15
```

其余的，如何恢复和恢复时修改index settings详见 [modules-snapshots.html#restore]( https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html#restore)



