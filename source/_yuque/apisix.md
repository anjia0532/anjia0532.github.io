---
title: 059-api网关之apache apisix初体验
urlname: apisix
date: 2021-03-10 20:35:21 +0800
tags: [openresty,nginx,kong,apisix,docker]
categories: [nginx]
---

> 这是坚持技术写作计划（含翻译）的第 59 篇，定个小目标 999，每周最少 2 篇。

本文简单讲解 apache apisix 和 apisix dashboard docker 安装及简单演示

<!-- more -->

## apisix 简单介绍

apisix 可能是 api 很 6 的的意思吧，哈哈
基于 nginx 系(nginx/openresty/tengine)的网关比较多，得益于 nginx/openresty/tengine 的高性能，在一些性能要求比较高的场景用 nginx 系的比较多

比较有名的当属开源的[kong](https://github.com/Kong/kong)和商业的 [openresty edge](https://openresty.com.cn/cn/)（openresty 创始人搞的商业版）,以及新秀 [apache apisix](https://github.com/apache/apisix)

关于 apisix 的特性和与 kong 的对比，详见 [官方文档](https://github.com/apache/apisix/blob/master/docs/zh/latest/README.md) ，就不复制粘贴了

## apisix 安装

apisix 支持容器化部署和裸机部署

关于 centos 和 ubuntu 的编译和安装，详见官方文档 [编译和安装](https://github.com/apache/apisix/blob/master/docs/zh/latest/README.md#%E7%BC%96%E8%AF%91%E5%92%8C%E5%AE%89%E8%A3%85)

关于容器化部署分两种，本地 quick start 的 [docker-compose](https://github.com/apache/apisix-docker/tree/master/example) 和 主流的 [k8s 的 helm-chart 安装 ](https://github.com/apache/apisix-helm-chart/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22)

## 快速启动

先跑个 demo，看看效果
我将 apisix-dashboard 合并到 apisix 中了，参考 [https://github.com/anjia0532/apisix-docker/commit/84c14d54745d33669555686bea93c00b2fb385ec](https://github.com/anjia0532/apisix-docker/commit/84c14d54745d33669555686bea93c00b2fb385ec)，也给[官方提 PR#150](https://github.com/apache/apisix-docker/pull/150)了（如果官方合并了后，建议用官方的库）

```bash
git clone https://github.com/anjia0532/apisix-docker.git
# 官方库
# git clone https://github.com/apache/apisix-docker.git
cd example
# 启动
docker-compose up -d

# 停止
docker-compose down
```

启动后，查看服务概览
[http://localhost:9080](http://localhost:9080) 是 apisix 的 http 端口, [http://localhost:9443](http://localhost:9443) 是 https 端口
localhost:2379 是 etcd 端口

```bash
docker-compose ps
           Name                         Command               State                       Ports
--------------------------------------------------------------------------------------------------------------------
example_apisix-dashboard_1   /usr/local/apisix-dashboar ...   Up      0.0.0.0:9000->9000/tcp
example_apisix_1             sh -c /usr/bin/apisix init ...   Up      0.0.0.0:9080->9080/tcp, 0.0.0.0:9443->9443/tcp
example_etcd_1               /entrypoint.sh etcd              Up      0.0.0.0:2379->2379/tcp, 2380/tcp
example_web1_1               /docker-entrypoint.sh ngin ...   Up      0.0.0.0:9081->80/tcp
example_web2_1               /docker-entrypoint.sh ngin ...   Up      0.0.0.0:9082->80/tcp

```

[http://localhost:9000](http://localhost:9000) 是 apisix-dashboard

用户名，密码都是 admin
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615280952922-96c06b75-8e35-4937-b94a-ee764fd831c6.png#align=left&display=inline&height=485&margin=%5Bobject%20Object%5D&name=image.png&originHeight=485&originWidth=621&size=25282&status=done&style=none&width=621)

通过 api 创建路由服务启停插件，可以参考 [官方文档-快速入门指南](https://apisix.apache.org/zh/docs/apisix/getting-started)

### 小试牛刀

此处简单示范一下，反代 [http://httpbin.org/](http://httpbin.org/)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615283010407-429f6e13-5408-46e4-8a07-5bcced8bad8d.png#align=left&display=inline&height=961&margin=%5Bobject%20Object%5D&name=image.png&originHeight=961&originWidth=1652&size=153448&status=done&style=none&width=1652)

### 创建消费者并启用插件

消费者，可以用于提供给下游的的 appid,比如用于限流，鉴权，开 zipkin 等操作，粒度到账号（注意这个账号是在 apisix 层面的，不涉及到后边应用）
开启 key-auth 插件
[官方文档-key-auth](https://apisix.apache.org/zh/docs/apisix/plugins/key-auth)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615286022584-156c65b2-792a-4c5d-a9ee-8746d98bc4e0.png#align=left&display=inline&height=529&margin=%5Bobject%20Object%5D&name=image.png&originHeight=529&originWidth=1473&size=70303&status=done&style=none&width=1473)
开启限流 limit-count
[官方文档-limit-count](https://apisix.apache.org/zh/docs/apisix/plugins/limit-count)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615286339189-b83b139e-30d1-44d7-b847-8bf313eb70ab.png#align=left&display=inline&height=891&margin=%5Bobject%20Object%5D&name=image.png&originHeight=891&originWidth=1345&size=89504&status=done&style=none&width=1345)
修改 httpbin_test 路由，启用 key-auth 插件
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615286554228-a0d0383b-b49f-48a6-a8d6-a04fa865f9a1.png#align=left&display=inline&height=564&margin=%5Bobject%20Object%5D&name=image.png&originHeight=564&originWidth=934&size=38600&status=done&style=none&width=934)
访问试一下 key-auth
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615286730648-41eb7ac1-278d-43ac-af0c-a1ed2b512c7d.png#align=left&display=inline&height=807&margin=%5Bobject%20Object%5D&name=image.png&originHeight=807&originWidth=1482&size=83760&status=done&style=none&width=1482)

试一下 limit-count,多刷新几次就会被拦截
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615286774576-259d57bc-1fdc-415c-a831-689f011ecbd0.png#align=left&display=inline&height=659&margin=%5Bobject%20Object%5D&name=image.png&originHeight=659&originWidth=1157&size=41849&status=done&style=none&width=1157)
启用 serverless 插件
[官方文档-serverless](https://apisix.apache.org/zh/docs/apisix/plugins/serverless)

配合`key-auth`演示一下`serverless-pre`插件，假设原有系统有自己的 header，而`apisix`的`key-auth`要求名字必须是`apikey`，改造原系统？还是自己抄一个`key-auth` ？工作量都很大，完全可以使用`serverless-pre`来做

摘抄官方文档的描述

> serverless 的插件有两个，分别是  `serverless-pre-function`  和  `serverless-post-function`， 前者会在指定阶段的最开始运行，后者是在指定阶段的最后运行。

serverless-pre 有两个参数，分别讲解下
phase: 是执行阶段，枚举值  ["rewrite", "access", "header_filter", "body_filter", "log", "balancer"]，借助 官方流程图会更好理解一点
![](https://raw.githubusercontent.com/apache/apisix/master/docs/assets/images/flow-plugin-internal.png#align=left&display=inline&height=683&margin=%5Bobject%20Object%5D&originHeight=683&originWidth=481&status=done&style=none&width=481)
functions：是自己写的匿名函数

示例

```javascript
{
  "functions": [
    "return function() ngx.req.set_header(\"apikey\", ngx.req.get_headers()[\"my-api-id\"]);  end"
  ],
  "phase": "rewrite"
}
```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615340302173-d7a1f140-eb4e-40b5-b0b7-c14e725ce233.png#align=left&display=inline&height=947&margin=%5Bobject%20Object%5D&name=image.png&originHeight=947&originWidth=1054&size=65961&status=done&style=none&width=1054)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615340942040-b3e555c1-09f4-4b3c-bd43-f5fc061639d8.png#align=left&display=inline&height=307&margin=%5Bobject%20Object%5D&name=image.png&originHeight=307&originWidth=674&size=15946&status=done&style=none&width=674)

## 批量创建路由/基于 openapi 导入

![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1615341054078-59d81f12-dafe-4274-b3f9-f7dd8e10848c.png#align=left&display=inline&height=539&margin=%5Bobject%20Object%5D&name=image.png&originHeight=539&originWidth=1435&size=76955&status=done&style=none&width=1435)
[官方文档-Import OpenAPI Guide](https://apisix.apache.org/zh/docs/dashboard/IMPORT_OPENAPI_USER_GUIDE)

## 自定义插件

[官方文档-自定义插件](https://apisix.apache.org/zh/docs/apisix/plugin-develop)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/03/10/apisix/)
- [我的掘金](https://juejin.cn/post/6937895404872663054/)
- [官方文档-快速入门指南](https://apisix.apache.org/zh/docs/apisix/getting-started)
