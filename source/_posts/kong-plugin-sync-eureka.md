
---

title: 048-使用Kong替换Zuul(从Eureka同步列表)

date: 2019-11-15 19:35:21 +0800

tags: [SpringCloud,zuul,eureka,kong,微服务]

categories: 微服务

---

> 这是坚持技术写作计划（含翻译）的第48篇，定个小目标999，每周最少2篇。


本文主要是介绍如何使用Kong替换Zuul作为SpringCloud的网关。并解决自动从Eureka同步服务实例到Kong的问题。

<!-- more -->

<a name="O5KGT"></a>
## 安装Kong 和 [kong-plugin-sync-eureka](https://github.com/anjia0532/kong-plugin-sync-eureka) 插件
以docker安装为例，详细请参考 [https://docs.konghq.com/install/docker/](https://docs.konghq.com/install/docker/)<br />创建 docker-compose.yml
```yaml
version: '2'
services:
  kong-plugins:
    image: kong:latest
    environment:
      KONG_LUA_PACKAGE_PATH: /usr/local/custom/share/lua/5.1/?.lua;;
    command:
    - luarocks
    - install
    - --tree
    - /usr/local/custom
    - kong-plugin-sync-eureka
    volumes:
    - /data/kong/custom/:/usr/local/custom/
  postgres:
    image: postgres:11-alpine
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: root
      POSTGRES_DB: kong
    volumes:
    - /data/postgresql/data:/var/lib/postgresql/data
  kong-migration:
    image: kong:latest
    environment:
      KONG_PG_HOST: postgres
      KONG_PG_PASSWORD: root
      KONG_PG_USER: root
      KONG_LUA_PACKAGE_PATH: /usr/local/custom/share/lua/5.1/?.lua;;
    command:
    - kong
    - migrations
    - bootstrap
    volumes:
    - /data/kong/custom/:/usr/local/custom/
    depends_on:
      - postgres
  kong:
    image: kong:latest
    environment:
      KONG_PG_HOST: postgres
      KONG_PG_PASSWORD: root
      KONG_PG_USER: root
      KONG_ADMIN_LISTEN: 0.0.0.0:9001
      KONG_PROXY_LISTEN: 0.0.0.0:9000
      KONG_PROXY_LISTEN_SSL: 0.0.0.0:9443
      KONG_LUA_PACKAGE_PATH: /usr/local/custom/share/lua/5.1/?.lua;;
      KONG_PLUGINS: bundled,sync-eureka
    volumes:
    - /data/kong/custom/:/usr/local/custom/
    - /data/kong/logs/:/usr/local/kong/logs/
    depends_on:
      - postgres
      - kong-migration
      - kong-plugins
    ports:
    - 9000:9000/tcp
    - 9001:9001/tcp
```

```bash
docker-compose ps
        Name                       Command               State                                            Ports                                          
--------------------------------------------------------------------------------------------------------------------------------------------------------
kong_kong-migration_1   /docker-entrypoint.sh kong ...   Exit 0                                                                                          
kong_kong-plugins_1     /docker-entrypoint.sh luar ...   Exit 0                                                                                          
kong_kong_1             /docker-entrypoint.sh kong ...   Up       8000/tcp, 8001/tcp, 8443/tcp, 8444/tcp, 0.0.0.0:9000->9000/tcp, 0.0.0.0:9001->9001/tcp 
kong_postgres_1         docker-entrypoint.sh postgres    Up       5432/tcp
```

<a name="VUoeN"></a>
## 启用插件

```bash
$ curl -H "Content-Type: application/json" -X POST  --data '{"config":{"sync_interval":10,"eureka_url":"http://eureka_server/eureka","clean_target_interval":86400},"name":"sync-eureka"}' http://localhost:9001/plugins
$ docker-compose restart kong
## 重启使插件生效
## 稍等10秒钟左右，访问services查看是否同步生效
$ curl http://localhost:9001/services
```



<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。<br />长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

<a name="35808e79"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/11/15/kong-plugin-sync-eureka/)
- [我的掘金](https://juejin.im/post/5dd25fcff265da0bbe51093f)
- [anjia0532/kong-plugin-sync-eureka](https://github.com/anjia0532/kong-plugin-sync-eureka)
- [Install Kong](https://konghq.com/install/)
- [Configuration Reference](https://docs.konghq.com/1.4.x/configuration/)

