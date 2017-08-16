---
title: openresty(nginx) redis 通用工具类
date: 2017-08-16 14:47:41
tags: [nginx,redis,lua,openresty,lua-resty-redis]
---

[openresty/lua-resty-redis][] 是章亦春开发的openresty中的操作redis的库。

截取官方部分代码，进行说明

```lua
    local redis = require "resty.redis"
    local red = redis:new()

    red:set_timeout(1000) -- 1 sec --设置超时时间

    local ok, err = red:connect("127.0.0.1", 6379) --设置redis的host和port
    if not ok then --判断生成连接是否失败
        ngx.say("failed to connect: ", err)
        return
    end

    ok, err = red:set("dog", "an animal") --插入键值(类似 mysql insert)
    if not ok then --判断操作是否成功
        ngx.say("failed to set dog: ", err)
        return
    end

    ngx.say("set result: ", ok) -- 页面输出结果
    -- put it into the connection pool of size 100,
    -- with 10 seconds max idle time
    local ok, err = red:set_keepalive(10000, 100) --将连接放入连接池,100个连接，最长10秒的闲置时间
    if not ok then --判断放池结果
        ngx.say("failed to set keepalive: ", err)
        return
    end
    -- 如果不放池，用完就关闭的话，用下面的写法
    -- or just close the connection right away:
    -- local ok, err = red:close()
    -- if not ok then
    --     ngx.say("failed to close: ", err)
    --     return
    -- end
```

如果用过java，c#等面向对象的语言，就会觉得这么写太。。。。了，必须重构啊，暴露太多无关细节了，导致代码中有大量重复代码了。

同样的内容，使用我封装后的代码。

```lua
    -- 依赖库
    local redis = require "resty.redis-util"
    -- 初始化
    local red = redis:new();
    -- 插入键值
    local ok,err = red:set("dog","an animal")
    -- 判断结果
    if not ok then
      ngx.say("failed to set dog:",err)
      return
    end
    -- 页面打印结果
    ngx.say("set result: ", ok) -- 页面输出结果
```

详细使用方法，参见我的项目 [anjia0532/lua-resty-redis-util][]

[openresty/lua-resty-redis]: https://github.com/openresty/lua-resty-redis
[anjia0532/lua-resty-redis-util]: https://github.com/anjia0532/lua-resty-redis-util
