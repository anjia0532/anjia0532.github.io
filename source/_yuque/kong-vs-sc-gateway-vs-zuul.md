---
title: 049-Kong1.4 vs SC Gateway2.2 vs Zuul1.3 性能测试
urlname: kong-vs-sc-gateway-vs-zuul
date: 2019-11-18 16:55:38 +0800
tags: []
categories: []
---

date: 2019-11-17 22:15:41
tags: [Openresty,Kong,Nginx,微服务,SpringCloud,zuul]
categories: [微服务]

---

> 这是坚持技术写作计划（含翻译）的第 49 篇，定个小目标 999，每周最少 2 篇。

本文主要对比常见 API 网关(kong 1.4,springcloud gateway 2.2, zuul 1.3)的性能测试（未涉及 service mesh、traefik 和 envoy）

<!-- more -->

## 下载测试代码

```bash
$ git clone https://github.com/anjia0532/gateway-kong-zuul.git
$ cd gateway-kong-zuul
$ mvn clean package
```

或者从 github 下载我打包好的

```bash
$ wget https://github.com/anjia0532/gateway-kong-zuul/releases/download/0.0.1-SNAPSHOT/sc-gateway-0.0.1-SNAPSHOT.jar
$ wget https://github.com/anjia0532/gateway-kong-zuul/releases/download/0.0.1-SNAPSHOT/web-0.0.1-SNAPSHOT.jar
$ wget https://github.com/anjia0532/gateway-kong-zuul/releases/download/0.0.1-SNAPSHOT/zuul-0.0.1-SNAPSHOT.jar
```

## endpoint

| endpoint                 | 描述                                          |
| ------------------------ | --------------------------------------------- |
| /api/ping                | 无入参，固定返回 "ok"                         |
| /api/echo?str=xx         | 入参 str,返回 str                             |
| /api/random-sleep?str=xx | 入参 str,随机 sleep100 毫秒-10 秒，并返回 str |

## 启动服务

| 命令                                    | 端口 | 描述               |
| --------------------------------------- | ---- | ------------------ |
| java -jar sc-gateway-0.0.1-SNAPSHOT.jar | 7080 | SpringCloudGateway |
|  |
| java -jar web-0.0.1-SNAPSHOT.jar        | 8080 | SpringMVC          |
|  |
| java -jar zuul-0.0.1-SNAPSHOT.jar       | 6080 | Zuul               |
|  |

| curl -i -X POST --url http://localhost:8001/services/ --data 'name=test' --data 'url=http://localhost:8080'

curl -i -X POST --url http://localhost:8001/services/test/routes --data 'paths[]=/test' | 9000 | Kong
http://localhost:8000/test/api/* |

## 安装  wrk

```bash
$ git clone https://github.com/wg/wrk
$ make
```

防止缓存，创建随机请求，注意第 6 行，在请求 8080，web，项目时，没有路由 `/test` 信息，

```bash
wrk.method = "GET";
wrk.body = "";

request = function()
        ip = tostring(math.random(1, 255)).."."..tostring(math.random(1, 255)).."."..tostring(math.random(1, 255)).."."..tostring(math.random(1, 255))
        path = "/test/api/echo?str=" .. ip
        return wrk.format(nil, path)
end
```

注意，我这是在 vagrant+virtualbox 里运行的,配置是 2U4G，详见 [Vagrantfile](https://github.com/Kong/kong-vagrant/blob/master/Vagrantfile)

```bash
## zuul (spring-cloud-starter-netflix-zuul 2.2.0.RC2 zuul-core 1.3.1)
./wrk -t10 -c10 -d60s -s ./test.lua --latency http://127.0.0.1:6080
Running 1m test @ http://127.0.0.1:6080
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     4.22ms    5.13ms 134.73ms   92.63%
    Req/Sec   292.26    116.64     1.37k    74.75%
  Latency Distribution
     50%    3.11ms
     75%    5.06ms
     90%    8.09ms
     99%   23.85ms
  174662 requests in 1.00m, 24.24MB read
Requests/sec:   2908.12
Transfer/sec:    413.22KB

## sc-gateway Spring Cloud Hoxton.RC2 (spring-cloud-gateway 2.2.0.RC2)
./wrk -t10 -c10 -d60s -s ./test.lua --latency http://127.0.0.1:7080
Running 1m test @ http://127.0.0.1:7080
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     3.11ms    2.53ms  55.45ms   84.39%
    Req/Sec   354.43    114.14     0.94k    68.56%
  Latency Distribution
     50%    2.56ms
     75%    4.01ms
     90%    5.88ms
     99%   12.12ms
  211742 requests in 1.00m, 26.11MB read
Requests/sec:   3526.95
Transfer/sec:    445.35KB

## Kong 1.4.0
./wrk -t10 -c10 -d60s -s ./test.lua --latency http://127.0.0.1:8000
Running 1m test @ http://127.0.0.1:8000
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.25ms    1.31ms  24.71ms   82.31%
    Req/Sec   461.43    140.31     1.76k    74.44%
  Latency Distribution
     50%    1.96ms
     75%    2.70ms
     90%    3.70ms
     99%    7.18ms
  275682 requests in 1.00m, 58.17MB read
Requests/sec:   4587.18
Transfer/sec:      0.97MB

## 直连 spring boot 2.2.1.RELEASE
./wrk -t10 -c10 -d60s -s ./test.lua --latency http://127.0.0.1:8080
Running 1m test @ http://127.0.0.1:8080
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     5.95ms   22.62ms 495.03ms   95.67%
    Req/Sec     1.22k   517.51     5.29k    69.52%
  Latency Distribution
     50%  543.00us
     75%    2.82ms
     90%   11.86ms
     99%  103.28ms
  714560 requests in 1.00m, 86.88MB read
Requests/sec:  11892.88
Transfer/sec:      1.45MB
```

运行三次，取最后一次的结果（除 web 项目一直启动外，其余随用随启，用完即停）

从 QPS 看   直连>Kong>Spring Cloud Gateway>Zuul

从内存和 CPU 看   直连肯定没有额外消耗，次之是 Kong，然后是 SC Gateway，最差的是 Zuul

从管理的易用性看，Kong 因为其便捷的插件体系及现有的插件生态完胜其余几样。

综合来看，建议使用 Kong 作为 API 网关。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2019/11/17/kong-vs-sc-gateway-vs-zuul/)
- [我的掘金](https://juejin.im/post/5dd26053f265da0bbe510940)
