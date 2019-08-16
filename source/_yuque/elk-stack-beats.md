---
title: ELK Stack之Beats简介
date: 2017-03-02 18:55:39
tags: [elk,elkstack,beats]
categories: [elkstasck]
---

Beats 是ELK Stack技术栈中负责单一用途数据采集并推送给Logstash或Elasticsearch的轻量级产品。
<!-- more -->
## Filebeat

Filebeat是一个轻量级日志收集工具。官网介绍说，当有几十，几百甚至上千台服务器、容器、虚拟机生成日志时，Filebeat提供一种轻量级简单的方式转发和收集日志。

####  健壮性
filebeat异常中断重启后会继续上次停止的位置。（通过${filebeat_home}\data\registry文件来记录日志的偏移量）
#### 智能调节传输速度，防止logstash、elasticsearch过载
Filebeat使用压力敏感协议(backpressure-sensitive)来传输数据，在logstash忙的时候，Filebeat会减慢读取-传输速度，一旦logsta恢复，则Filebeat恢复原来的速度。
![Filebeat](https://static-www.elastic.co/assets/blt203883a0718cdc5a/filebeat-diagram.png?q=891)

## Metricbeat

Metricbeat是一个轻量级的系统级性能指标监控工具。收集CPU，内存，磁盘等系统指标和Redis，nginx等各种服务的指标

#### 简化系统监控

通过在Linux，Windows，Mac上部署Metricbeat，可以收集cpu，内存，文件系统，磁盘IO，网络IO等统计信息
![简化系统监控](https://static-www.elastic.co/assets/bltcaef5fe4015417d6/product-metricbeat-1.gif?q=891)

#### 多模块监控支持
支持采集Apache, NGINX, MongoDB, MySQL, PostgreSQL, Redis, and ZooKeeper等服务的指标。零依赖，只需要在配置文件中启用即可
![模块监控](https://static-www.elastic.co/assets/blt53458fcf0602cad1/metricbeat-logos-4.svg?q=891)

#### 监控容器
如果你使用Docker管理你的服务。可以在该主机上单独起一个Metricbeat容器，他通过从proc文件系统中直接读取cgroups信息来收集有关Docker主机上每个容器的统计信息。不需要特殊权限访问Docker API

#### 无缝接入ELK
Metricbeats是ELK Stack全家桶中的一员，可以和ELK无缝协同工作。例如使用Logstash二次处理数据，用Elasticsearch分析，或者用Kibana创建和共享仪表盘。

## Packetbeat

Packetbeat是一个轻量级的网络数据包分析工具。如果你用过wireshark，fiddler会很好理解数据包分析的概念，如果没用过，那你可以参考Chrome的dev tools的Network的功能。Packetbeat可以通过抓包分析应用程序的网络交互。并且将抓到的数据发送到Logstash或者Elasticsearch。

#### 实时监控你的服务和应用程序

Packetbeat 轻松的实时监控并解析像HTTP这样的网络协议。以了解流量是如何经过你的网络。Packetbeat是被动的，不增加延迟开销，无代码侵入。不干涉其他基础设施

#### 支持多种应用层协议

Packetbeat是一个库，支持多种应用程序层协议，如下所示
![支持多种应用层协议](https://static-www.elastic.co/assets/bltbc6a3f306a97dd20/packetbeat-logo-4.svg?q=891)

#### 可以搜索和分析网络流量
Packetbeat可以让你实时在目标服务器上进行抓包-解码-获取请求和响应-展开字段-将json格式的结果发送到Elasticsearch
![搜索和分析网络流量](https://static-www.elastic.co/assets/bltd9ea7151b811b1c6/packetbeat-monitoring-steps3.svg?q=891)

#### 无缝接入ELK
Packetbeat是ELK Stack全家桶中的一员，可以和ELK无缝协同工作。例如使用Logstash二次处理数据，用Elasticsearch分析，或者用Kibana创建和共享仪表盘。

## Winlogbeat

Winlogbeat是一个轻量级的Windows事件日志收集工具。将Windows事件发送到Elasticsearch或者Logstash

#### 从任何Windows事件日志通道(Channel)读取

如果你有Windows服务器的话，其实可以从Windows事件日志中看到很多东西。例如，登陆(4624),登陆失败(4625),插入USB便携设备(4663)或者新装软件(11707)。WinlogBeat可以配置从任何事件日志通道读取并且结构化提供原始事件数据。使得通过Elasticsearch过滤和聚合结果变得很容易。
![从任何Windows事件日志通道(Channel)读取](https://static-www.elastic.co/assets/blt1d5fc2155a06db09/winlogbeat-diagram.jpg?q=891)

#### 无缝接入ELK
Winlogbeat是ELK Stack全家桶中的一员，可以和ELK无缝协同工作。例如使用Logstash二次处理数据，用Elasticsearch分析，或者用Kibana创建和共享仪表盘。

## Heartbeat

Heartbeat 是一个心跳检测工具，主要监控服务的可用性。监控给定的地址是否可用(官网原话：对于给定的URL列表，**Heartbeat就问一句，还活着没？活着吱一声。。。**) 可以结合ELK Stack其他产品做进一步的分析

#### 容易上手，配置简单

不管你是测试同主机服务还是其他网络服务，Heartbeat都可以很轻松的生成正常运行时间和响应时间数据。而且修改配置不需要重启Heartbeat

#### Ping你想Ping的任何东西

Heartbeat通过ICMP,TCP,和HTTP进行ping，也支持TLS，身份验证（authentication ），和代理(proxies)。由于简单的DNS解析，你可以监控所有负载均衡的服务(原文:You can monitor all the hosts behind a load-balanced server thanks to simple DNS resolution)

#### 动态添加和删除目标

现如今基础设施，服务和主机经常动态调整。Heartbeat可以修改配置文件后自动加载(原文:Heartbeat makes it easy to automate the process of adding and removing monitoring targets via a simple, file-based interface.)

#### 无缝接入ELK

Heartbeat是ELK Stack全家桶中的一员，可以和ELK无缝协同工作。例如使用Logstash二次处理数据，用Elasticsearch分析，或者用Kibana创建和共享仪表盘。