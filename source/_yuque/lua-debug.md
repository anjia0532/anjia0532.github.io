---
title: '078-Lua远程调试(以Apisix,Redis为例)'
urlname: lua-debug
date: '2022-08-29 19:35:21 +0800'
tags:
  - apisix
  - nginx
  - openresty
  - kong
  - redis
  - lua
categories:
  - lua
---

> 这是坚持技术写作计划（含翻译）的第 78 篇，定个小目标 999，每周最少 2 篇。

Lua 是一门轻量级类 C 高性能脚本语言，广泛用于 游戏领域，Redis 自定义脚本，Nginx 系(openresty,kong,apisix 等)，wireshark 的脚本，wrk 压测的脚本，路由器(比如 openwrt ), 物联网领域（比如 nodemcu,PlatformIO,OpenMQTTGateway,tasmota 等）,游戏领域没有涉猎。其他列出的方面都多多少少接触过，写过 lua 代码。

lua 的语法教程不是本文的重点，可以网上搜索，比如 [LUA 简明教程](https://coolshell.cn/articles/10739.html)

本文主要以 Apisix(Nginx 系) 和 Redis 两个为例，讲解如何进行断点调试，而不是 Print 大法来逮 bug。

<!-- more -->

## 安装 Redis

如果是 Mac/Linux 那可以去 [https://redis.io/download/#redis-stack-downloads](https://redis.io/download/#redis-stack-downloads) 下载对应的系统，或者从源代码编译。

如果是 Windows ,并且是 Windows 10+，可以用 windows 子系统功能，安装 Linux 版本 Redis。或者在 Windows 上安装 Docker。

如果是 低版本 Windows。只能安装网上提供的低版本(好像是 redis 3 和 4) 的非官方版本的 Redis（最新 GA 是 6.x 预览版是 7.x）

本文一律采用 Docker 方式进行演示。对于 Docker 基础知识不做介绍，不了解可以去网上搜索相关内容。

```shell
# docker run --name redis-6-demo -p6379:6379 -d redis:6-alpine redis-server

# docker ps
CONTAINER ID   IMAGE            COMMAND                  CREATED          STATUS          PORTS                                       NAMES
713c1863c29e   redis:6-alpine   "docker-entrypoint.s…"   53 seconds ago   Up 52 seconds   0.0.0.0:6379->6379/tcp,</div>6379->6379/tcp   redis-6-demo

```

常用 Redis GUI 工具

- [https://redis.com/redis-enterprise/redis-insight/](https://redis.com/redis-enterprise/redis-insight/)[](https://github.com/qishibo/AnotherRedisDesktopManager)
- [https://github.com/uglide/RedisDesktopManager](https://github.com/uglide/RedisDesktopManager)
- [https://github.com/qishibo/AnotherRedisDesktopManager](https://github.com/qishibo/AnotherRedisDesktopManager)

Redis 官方也有调试工具 LDB，参考 [Debugging Lua scripts in Redis](https://redis.io/docs/manual/programmability/lua-debugging/)

Redis 为啥要用 Lua

- 减少网络开销。可以将多个请求通过脚本的形式一次发送，减少网络时延。
- 原子操作。Redis 会将整个脚本作为一个整体执行，中间不会被其他请求插入。因此在脚本运行过程中无需担心会出现竞态条件，无需使用事务。
- 复用。客户端发送的脚本会永久存在 redis 中，这样其他客户端可以复用这一脚本，而不需要使用代码完成相同的逻辑。

Java 系的 Redis 库 [redisson/redisson](https://github.com/redisson/redisson/wiki/Table-of-Content) 大量用了 Lua ，可以参考，借鉴一下。

## 安装并配置 ZeroBrane Studio

下载 [ZeroBrane Studio](https://studio.zerobrane.com/download?not-this-time) 到本地并解压。

下载 [ZeroBranePackage/master/redis.lua](https://raw.githubusercontent.com/pkulchenko/ZeroBranePackage/master/redis.lua) 并保存到 `/path/to/zerobranestudio/packages/` 目录下。

启动 ZB Studio -> Project -> Lua Interpreter -> Redis
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1661508064183-d46d69d8-36ae-422e-a67a-3d87d7dadd71.png#clientId=u9757d729-3f06-4&from=paste&height=730&id=uaf921b57&originHeight=730&originWidth=693&originalType=binary∶=1&rotation=0&showTitle=false&size=51956&status=done&style=none&taskId=ucdf7b00c-da07-4f68-86b1-1118b274547&title=&width=693)
创建 `demo.lua`

```lua
local key = KEYS[1]
local val = ARGV[1]

redis.call("set", key, val)

val = redis.call("get", key)

return val
```

### 使用 ZB Studio 调试 Redis Lua Scripts

![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1661740417584-4b0aeb91-8d2e-4c70-8280-20e89c45d3e3.png#clientId=u1fbb65af-af0d-4&from=paste&height=736&id=u547b5fe0&originHeight=736&originWidth=407&originalType=binary∶=1&rotation=0&showTitle=false&size=58457&status=done&style=none&taskId=u11280140-6ef5-467f-a0e4-c163bab9ee6&title=&width=407)
在 `Project` -> `Command Line Parameters...` 添加参数，注意参数用英文`,`分隔，前后加空格,也就是 `key空格,空格value`

点击 `Project `-> `Start Debugging` 或者直接按快进键 `F5` 启动调试。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1661741017793-63a69c19-61d3-44a6-ba83-e627a64dce63.png#clientId=u1fbb65af-af0d-4&from=paste&height=268&id=ubff51ec1&originHeight=268&originWidth=502&originalType=binary∶=1&rotation=0&showTitle=false&size=63859&status=done&style=none&taskId=uebf9c098-d916-4f13-945b-e93063aa8bc&title=&width=502)
`Step Into(F10)` 没有函数时跟 `Step Over(Shift + F10)` 一样，都是执行下一步。区别是，遇到函数时，Step Into 会进入函数内部，Step Over 会将函数当成一步.

## 安装 Apisix

为了简化步骤，本文以 docker-compose 安装为例。
文件结构如下所示

```
.
├── config.yaml
├── docker-compose.yaml
└── plugins
    └── usr
        └── local
            └── apisix
                └── apisix
                    └── plugins
                        ├── example.lua
                        └── mobdebug.lua
```

下载 [https://raw.githubusercontent.com/pkulchenko/MobDebug/master/src/mobdebug.lua](https://raw.githubusercontent.com/pkulchenko/MobDebug/master/src/mobdebug.lua) 保存到 `./plugins/usr/local/apisix/apisix/plugins`文件夹下

`config.yaml`

```yaml
apisix:
  node_listen: 9080 # APISIX listening port
  enable_ipv6: false

  allow_admin: # http://nginx.org/en/docs/http/ngx_http_access_module.html#allow
    - 0.0.0.0/0 # We need to restrict ip access rules for security. 0.0.0.0/0 is for test.

  admin_key:
    - name: "admin"
      key: edd1c9f034335f136f87ad84b625c8f1
      role:
        admin # admin: manage all configuration data
        # viewer: only can view configuration data
    - name: "viewer"
      key: 4054f7cf07e344346cd3f287985e76a2
      role: viewer

  enable_control: true
  control:
    ip: "0.0.0.0"
    port: 9092

etcd:
  host: # it's possible to define multiple etcd hosts addresses of the same etcd cluster.
    - "http://etcd:2379" # multiple etcd address
  prefix: "/apisix" # apisix configurations prefix
  timeout: 30 # 30 seconds

plugins:
  - example
```

`example.lua`

```lua
local plugin_name = "example"
local core = require("apisix.core")
local mobdebug = require("apisix.plugins.mobdebug")


local schema = {
    type = "object",
    properties = {}
}

local _M = {
    version = 0.1,
    priority = 0,
    name = plugin_name,
    schema = schema
}

function _M.check_schema(conf)
    return core.schema.check(schema, conf)
end

function _M.access(conf, ctx)
    -- 把 192.168.xx.xxx 换成 idea 运行主机 ip
    mobdebug.start("192.168.xx.xxx", 8172)
    core.log.warn(core.json.encode(conf))
end

return _M

```

注意更换 `192.168.xx.xxx`为实际 ip
更多 Apisix 插件开发信息，详见 [插件开发](https://apisix.apache.org/zh/docs/apisix/plugin-develop/)

`docker-compose.yaml`

```yaml
version: "3"

services:
  apisix:
    image: apache/apisix:2.15.0-alpine
    restart: always
    volumes:
      - ./config.yaml:/usr/local/apisix/conf/config.yaml:ro
      - ./plugins/usr/local/apisix/apisix/plugins/example.lua:/usr/local/apisix/apisix/plugins/example.lua:ro
      - ./plugins/usr/local/apisix/apisix/plugins/mobdebug.lua:/usr/local/apisix/apisix/plugins/mobdebug.lua:ro
    depends_on:
      - etcd
    ports:
      - "9080:9080/tcp"
      - "9091:9091/tcp"
      - "9443:9443/tcp"
      - "9092:9092/tcp"
    networks:
      apisix:

  etcd:
    image: bitnami/etcd:3.4.9
    restart: always
    environment:
      ETCD_ENABLE_V2: "true"
      ALLOW_NONE_AUTHENTICATION: "yes"
      ETCD_ADVERTISE_CLIENT_URLS: "http://0.0.0.0:2379"
      ETCD_LISTEN_CLIENT_URLS: "http://0.0.0.0:2379"
    ports:
      - "2379:2379/tcp"
    networks:
      apisix:

  web1:
    image: nginx:1.18.0-alpine
    restart: always
    ports:
      - "9081:80/tcp"
    environment:
      - NGINX_PORT=80
    networks:
      apisix:

networks:
  apisix:
    driver: bridge
```

- 运行 `docker-compose up -d `
- 查看 apisix 日志`docker-compose logs -f apisix`
- 重启 apisix`docker-compose restart apisix`
- 只重新加载配置或者 lua 文件，不重启 apisix `docker-compose exec apisix apisix reload`

```bash
# 新增路由
curl --location --request PUT 'http://127.0.0.1:9080/apisix/admin/routes/1' \
--header 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "methods": ["GET"],
    "uri": "/index.html",
    "id": 1,
    "plugins": {
        "example": {
        }
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "web1:80": 1
        }
    }
}'
# 访问路由，测试断点是否生效
curl --location --request GET 'http://127.0.0.1:9080/index.html' \
--header 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' \
--header 'Content-Type: application/json'
```

### 通过 EmmyLua 调试 Apisix 插件

右键 `plugins` 文件夹，将目录标记为 `源代码根目录`
将要加的断点加上
在他之前加上 `mobdebug.start("192.168.xx.xxx", 8172)`
访问路由，正常情况下，就会进断点。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1661842774972-de508543-5f89-41e6-84ac-6d0386a8a532.png#clientId=u82e64653-60a4-4&from=paste&height=779&id=ub2209751&originHeight=779&originWidth=1125&originalType=binary∶=1&rotation=0&showTitle=false&size=328262&status=done&style=none&taskId=u6c54b698-2903-428f-a994-76b63804f5e&title=&width=1125)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1661828017104-12be9640-41a1-4c22-b79e-8c8ad1bf1b37.png#clientId=ufb31683a-9ead-4&from=paste&height=651&id=ufb45076d&originHeight=651&originWidth=764&originalType=binary∶=1&rotation=0&showTitle=false&size=340744&status=done&style=none&taskId=u304f418a-b243-4581-b752-ee7cb153cb3&title=&width=764)
注意:

- idea 里将某个文件夹设置成源代码根目录后，其子目录的 lua 文件要与被调试的环境下的 lua 文件的路径完全一致。比如 docker 里，apisix 插件目录的绝对路径是 `/usr/local/apisix/apisix/plugins`,那就要求 `plugins` 目录下对应创建这个层级关系才行。不然 mobdebug 不走断点。
- 调试时 `mobdebug.start(ip,端口)`其实是连接 emmylua 插件，或者 ZB Studio 的 DebugServer 开放的端口，此处要保持一致。可以在 apisix 所在主机用 telnet ip 端口 试试能否通信。
- 执行了 restart apisix 或者 apisix reload 后，client 端连接断开了，但是 idea 端还没释放，再次访问会报错，所以需要重新点击调试按钮。
- 不建议生产环境下用 mobdebug 进行调试。

### 其他

此种方法也可以用 ZeroBrane Studio 作为 debug server 端,进行调试，而不用 emmylua 插件。 可以参考 [lua 学习笔记（4）-- 搭建 mobdebug 远程开发环境](https://blog.csdn.net/weixin_41534481/article/details/125107411)

如果是在本机调试(不是本机跑 docker)，也可以试试 EmmyLua 的 [Attach Debug 附加调试](https://emmylua.github.io/zh_CN/run/attach.html)。

如果是本机调试 Apisix 自身代码，可以参考 [APISIX Runtime Debug/动态调试](https://juejin.cn/post/6951650129044570125) 写的挺详细了，不再赘述。

结合 [077-加快云原生应用开发速度(Nocalhost 篇)](https://anjia0532.github.io/2022/08/19/k8s-nocalhost/) 是可以调试通过 [helm 安装到 k8s 的 apisix](https://github.com/apache/apisix-helm-chart/tree/master/charts/apisix) 的。具体原理上篇已经写的挺详细了，不再赘述。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/08/29/lua-debug/)
- [我的掘金](https://juejin.cn/post/7137502232253054989/)[](https://juejin.cn/post/6951650129044570125)
- [pkulchenko/MobDebug](https://github.com/pkulchenko/MobDebug)
- [EmmyLua Remote Debug 远程调试](https://emmylua.github.io/zh_CN/run/remote.html)
- [APISIX Runtime Debug/动态调试](https://juejin.cn/post/6951650129044570125)[](https://emmylua.github.io/zh_CN/run/remote.html)
- [关于 Lua 无法调试总结。](https://forum.cocos.org/t/lua/78205)
- [ZeroBrane Studio](https://studio.zerobrane.com/)
- [ZeroBrane Studio - ZeroBrane Studio Plugin for Redis Lua Scripts](http://larrynung.github.io/2017/04/18/ZeroBrane-Studio-ZeroBrane-Studio-Plugin-for-Redis-Lua-Scripts/)
- [ZeroBrane Studio Plugin for Redis Lua Scripts](https://redis.com/blog/zerobrane-studio-plugin-for-redis-lua-scripts/)
- [EmmyLua MobDebug 浅析](https://zhuanlan.zhihu.com/p/64362155)
