---
title: 068-用go写个同步eureka/nacos实例到apisix/kong的工具
urlname: discovery-syncer
date: '2021-12-30 20:35:21 +0800'
tags:
  - apisix
  - kong
  - nginx
  - golang
  - eureka
  - nacos
  - 微服务
categories:
  - nginx
---

> 这是坚持技术写作计划（含翻译）的第 68 篇，定个小目标 999，每周最少 2 篇。

用业余时间，一边百度 golang 语法一边写了个从 eureka/nacos 这类注册中心同步实例信息到 apisix/kong 的 upstream 的工具。

<!-- more -->

## 项目介绍

项目地址 [anjia0532/discovery-syncer](https://github.com/anjia0532/discovery-syncer)

支持从 nacos(已实现)，eureka(已实现)等注册中心同步到 apisix(已实现)和 kong(已实现)
等网关，后续将支持自定义插件，支持用户自己用 golang 实现支持类似携程阿波罗注册中心，etcd 注册中心，consul 注册中心等插件，以及 spring gateway 等网关插件的高扩展性

## 快速开始

### 通过二进制运行

从 [releases](https://github.com/anjia0532/discovery-syncer/releases) 下载最新的对应系统的二进制文件

```bash

discovery-syncer-windows-amd64.exe --help

usage: discovery-syncer-windows-amd64.exe [<flags>]

Flags:
  -h, --help  Show context-sensitive help (also try --help-long and --help-man).
  -p, --web.listen-address=":8080"
              The address to listen on for web interface.
  -c, --config.file="config.yml"
              Path to configuration file.
```

### 通过 docker 运行

```bash
docker run anjia0532/discovery-syncer:v1.0.4
```

特别的，`-c` 支持配置远端 http[s]的地址，比如读取静态资源的，比如读取 nacos 的 `-c http://xxxxx/nacos/v1/cs/configs?tenant=public&group=DEFAULT_GROUP&dataId=discovery-syncer.yaml`,便于管理

### 配置文件

```yaml
# 是否启用 pprof
# 通过 http://ip:port/debug/pprof/ 访问
enable-pprof: false
logger:
  level: debug # debug,info,error
  logger: console # console vs file
  log-file: syncer.log # The file name of the logger output, does not exist automatically
  date-slice: y # Cut the document by date, support "y" (year), "m" (month), "d" (day), "h" (hour), default "y".

# 注册中心,map形式
discovery-servers:
  # nacos1 是注册中心的名字，可以随便定义，但是不能重复
  nacos1:
    # 类型，目前仅支持 nacos和eureka
    type: nacos
    # 默认，如果注册中心没有返回权重时，添加的默认权重
    weight: 100
    # 注册中心的url前缀
    prefix: /nacos/v1/
    # 注册中心的连接地址，注意最后不能带/
    host: "http://nacos-server:8858"
  eureka1:
    type: eureka
    weight: 100
    prefix: /eureka/
    # 对于basic认证，可以这么写
    host: "http://admin:admin@eureka-server:8761"

# 网关,map形式
gateway-servers:
  # 网关名字，可以随便写，但是不能重复
  apisix1:
    # 网关类型，目前支持apisix和kong
    type: apisix
    # 管理端host,注意最后不能有/
    admin-url: http://apisix-server:9080
    # 管理端uri前缀
    prefix: /apisix/admin/
    # 特别的扩展参数，在config里用key:value形式添加
    config:
      X-API-KEY: xxxxx
  kong1:
    type: kong
    admin-url: http://kong-server:8001
    prefix: /upstreams/

# 同步任务，列表形式
targets:
  # 注意同一个注册中心，但是有多个租户（类似nacos的命名空间）时，不需要创建多个相同的注册中心
  # 只需要创建多个targets，然后改config的扩展参数即可
  # 第一个任务
  # 注册中心，来源
  - discovery: nacos1
    # 网关，目标
    gateway: apisix1
    # 是否启用
    enabled: false
    # 拉取间隔，具体支持表达式，详见 https://github.com/robfig/cron
    fetch-interval: "@every 10s"
    # 默认是把能拉倒的注册中心的服务都拉过来，有些不需要的，则进行排除,支持正则
    exclude-service: ["ex*", "test"]
    # 同步到网关的upstream的名字的前缀，便于管理
    upstream-prefix: nacos1
    # 对于health检查时，超过限定秒数的，认为是失联状态，默认是10秒
    maximum-interval-sec: 20
    # 扩展参数
    config:
      # nacos 的groupName
      groupName: DEFAULT_GROUP
      # nacos的 namespace
      namespaceId: test
      # 创建到apisix 的upstream的默认模板，具体支持的模板语法，自行搜索 golang text/template
      template: |
        {
            "id": "syncer-{{.Name}}",
            "timeout": {
                "connect": 30,
                "send": 30,
                "read": 30
            },
            "name": "{{.Name}}",
            "nodes": {{.Nodes}},
            "type":"roundrobin",
            "desc": "auto sync by https://github.com/anjia0532/discovery-syncer"
        }

  - discovery: eureka1
    gateway: kong1
    enabled: false
    fetch-interval: "@every 5s"
    maximum-interval-sec: 10
    config:
      template: |
        {
            "name": "{{.Name}}",
            "algorithm": "round-robin",
            "hash_on": "none",
            "hash_fallback": "none",
            "hash_on_cookie_path": "/",
            "slots": 10000,
            "healthchecks": {
                "passive": {
                    "healthy": {
                        "http_statuses": [200, 201, 202, 203, 204, 205, 206, 207, 208, 226, 300, 301, 302, 303, 304, 305, 306, 307, 308],
                        "successes": 0
                    },
                    "type": "http",
                    "unhealthy": {
                        "http_statuses": [429, 500, 503],
                        "timeouts": 0,
                        "http_failures": 0,
                        "tcp_failures": 0
                    }
                },
                "active": {
                    "timeout": 1,
                    "https_sni": "example.com",
                    "http_path": "/",
                    "concurrency": 10,
                    "https_verify_certificate": true,
                    "type": "http",
                    "healthy": {
                        "http_statuses": [200, 302],
                        "successes": 0,
                        "interval": 0
                    },
                    "unhealthy": {
                        "http_statuses": [429, 404, 500, 501, 502, 503, 504, 505],
                        "timeouts": 0,
                        "http_failures": 0,
                        "interval": 0,
                        "tcp_failures": 0
                    }
                },
                "threshold": 0
            },
            "tags": ["discovery-syncer-auto"]
        }
```

### Api 接口

| 路径                    | 返回值 | 用途                                                                                     |
| ----------------------- | ------ | ---------------------------------------------------------------------------------------- |
| `GET /`                 | `OK`   | 服务是否启动                                                                             |
| `GET /-/reload`         | `OK`   | 重新加载配置文件，加载成功返回 OK，主要是 cicd 场景或者 k8s 的 configmap reload 场景使用 |
| `GET /health`           | JSON   | 判断服务是否健康，可以配合 k8s 等容器服务的健康检查使用                                  |
| `PUT /discovery/{name}` | `OK`   | 主动下线上线注册中心的服务,配合 CICD 发版业务用                                          |

`GET /health` 的返回值

```json
{
  // 一共有几个enabled的同步任务(targets)
  "total": 2,
  // 正常在跑的有几个
  "running": 2,
  // 有几个超过 配置文件定义的maximum-interval-sec的检测时间没有运行的，失联的。
  "lost": 0,
  // 都在跑，状态是OK（http状态码是200），有在跑的，有失联的，状态是WARN（http状态码是200），全部失联，状态是DOWN(http状态码500)
  "status": "OK",
  // 哪些成功，哪些失败
  "details": ["syncer:a_task,is ok", "syncer:b-api,is ok"],
  // 运行时长
  "uptime": "1m6s"
}
```

`PUT /discovery/{name}` 中的 name 是注册中的名字，如果不存在，则返回 `Not Found`

body 入参

```json
{
  // 检索哪个服务下的实例
  "serviceName": "",
  // 基于注册中心元数据还是基于实例ip来查找
  "type": "METADATA/IP",
  // 匹配的查询条件，支持正则
  "regexpStr": "",
  // 匹配的元数据key，如果是ip则不用填
  "metadataKey": "",
  // 匹配到的将状态改成上线还是下线
  "status": "UP/DOWN",
  // 其他没匹配的，状态是上线还是下线，ORIGIN保持不变
  "otherStatus": "UP/DOWN/ORIGIN",
  // 扩展参数
  "extData": {}
}
```

## 待优化点

1.  目前的同步任务是串行的，如果待同步的量比较大，或者同步时间窗口设置的特别小的情况下，会导致挤压
2.  不支持自定义同步插件，不利于自行扩展
3.  同步机制目前是基于定时轮询，效率比较低，有待优化，比如增加缓存开关，上游注册中心与缓存比对没有差异的情况下，不去拉取/变更下游网关的 upstream 信息，或者看看注册中心支不支持变动主动通知机制等。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/12/30/discovery-syncer/)
- [我的掘金](https://juejin.cn/post/7047670699762122760/)
