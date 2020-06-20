---
title: 031-数据可视化之redash(支持43种数据源)
urlname: redash
date: 2019-07-08 12:30:00 +0800
tags: [CDH,hadoop,大数据,数据可视化,Superset,redash]
categories: [大,数,据]
---

> 这是坚持技术写作计划（含翻译）的第 31 篇，定个小目标 999，每周最少 2 篇。

本文是数据可视化系列第二篇,本系列会讲解  [PowerBI/Excel](https://juejin.im/post/5cdfdfa4f265da1bbf68ec4b),[Metabase](https://www.metabase.com/),[Redash](https://anjia0532.github.io/2019/07/08/redash),[Superset](http://superset.apache.org/),[CBoard](https://juejin.im/post/5b4ee1c2f265da0f5d4cc978)

人类都是视觉动物，讲究一图胜千言。如果没了可视化，那么你在跟领导汇报工作时，很大程度会鸡同鸭讲。
其实 excel2016+已经是一个不错的数据分析及可视化工具了(支持几十种数据源),但是，不方便权限控制，集中，及报警。

我一般将 redash 作为可视化工具、数据库查询编辑器(类似 navicat-premium)、数据挖掘探索工具来用。
截止目前，自建 redash 支持 43 种数据源
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562561680709-399441ab-b14d-4d93-b797-60925f2eafd6.png#align=left&display=inline&height=944&name=image.png&originHeight=944&originWidth=1133&size=147027&status=done&width=1133)

<!-- more -->

## 安装 redash

```bash
## 安装必要工具
apt install -y pwgen python-pip
pip install pip -U
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
pip install docker-compose

## 生成脚本
cat << EOF | sudo tee -a ./setup.sh
#!/usr/bin/env bash
# This script setups dockerized Redash on Ubuntu 18.04.
set -eu

REDASH_BASE_PATH=/opt/redash

create_directories() {
    if [[ ! -e $REDASH_BASE_PATH ]]; then
        sudo mkdir -p $REDASH_BASE_PATH
        sudo chown $USER:$USER $REDASH_BASE_PATH
    fi

    if [[ ! -e $REDASH_BASE_PATH/postgres-data ]]; then
        mkdir $REDASH_BASE_PATH/postgres-data
    fi
}

create_config() {
    if [[ -e $REDASH_BASE_PATH/env ]]; then
        rm $REDASH_BASE_PATH/env
        touch $REDASH_BASE_PATH/env
    fi

    COOKIE_SECRET=$(pwgen -1s 32)
    SECRET_KEY=$(pwgen -1s 32)
    POSTGRES_PASSWORD=$(pwgen -1s 32)
    REDASH_DATABASE_URL="postgresql://postgres:${POSTGRES_PASSWORD}@postgres/postgres"

    echo "PYTHONUNBUFFERED=0" >> $REDASH_BASE_PATH/env
    echo "REDASH_LOG_LEVEL=INFO" >> $REDASH_BASE_PATH/env
    echo "REDASH_REDIS_URL=redis://redis:6379/0" >> $REDASH_BASE_PATH/env
    echo "POSTGRES_PASSWORD=$POSTGRES_PASSWORD" >> $REDASH_BASE_PATH/env
    echo "REDASH_COOKIE_SECRET=$COOKIE_SECRET" >> $REDASH_BASE_PATH/env
    echo "REDASH_SECRET_KEY=$SECRET_KEY" >> $REDASH_BASE_PATH/env
    echo "REDASH_DATABASE_URL=$REDASH_DATABASE_URL" >> $REDASH_BASE_PATH/env
}

create_directories
create_config
EOF

## 生成必要配置文件
chmod +x ./setup && ./setup
```

docker-compose.yml

```yaml
version: "2"
x-redash-service: &redash-service
  image: redash/redash:7.0.0.b18042
  depends_on:
    - postgres
    - redis
  env_file: /opt/redash/env
  restart: always
services:
  server:
    <<: *redash-service
    command: server
    ports:
      - "5000:5000"
    environment:
      REDASH_WEB_WORKERS: 4
  scheduler:
    <<: *redash-service
    command: scheduler
    environment:
      QUEUES: "celery"
      WORKERS_COUNT: 1
  scheduled_worker:
    <<: *redash-service
    command: worker
    environment:
      QUEUES: "scheduled_queries,schemas"
      WORKERS_COUNT: 1
  adhoc_worker:
    <<: *redash-service
    command: worker
    environment:
      QUEUES: "queries"
      WORKERS_COUNT: 2
  redis:
    image: redis:5.0-alpine
    restart: always
  postgres:
    image: postgres:9.5-alpine
    env_file: /opt/redash/env
    volumes:
      - /opt/redash/postgres-data:/var/lib/postgresql/data
    restart: always
  nginx:
    image: redash/nginx:latest
    ports:
      - "80:80"
    depends_on:
      - server
    links:
      - server:redash
    restart: always
```

```bash
## 配置数据库
sudo docker-compose run --rm server create_db
## 启动
sudo docker-compose up -d
```

## 配置 redash

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562631840989-c6c2d739-b9df-4b95-a3fa-61cdc7b35dde.png#align=left&display=inline&height=626&name=image.png&originHeight=626&originWidth=508&size=35369&status=done&width=508)

创建数据源
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562631903553-80df8657-983c-4d8e-b7a1-d1c6e126fb62.png#align=left&display=inline&height=620&name=image.png&originHeight=620&originWidth=1328&size=93814&status=done&width=1328)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562634341179-6242da8f-b79e-424c-bd1c-4ef8d959b6f2.png#align=left&display=inline&height=616&name=image.png&originHeight=616&originWidth=965&size=35160&status=done&width=965)

> 注意：
> 为做演示，clickhouse 已导入官网提供的 2018 年航天数据，详见  [https://clickhouse.yandex/docs/zh/getting_started/example_datasets/ontime/](https://clickhouse.yandex/docs/zh/getting_started/example_datasets/ontime/)

## 演示 redash

创建查询 **查询 2007 年各航空公司延误超过 10 分钟以上的百分比** 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736509902-041060bf-8e40-4025-82a0-cb9b6d2accaf.png#align=left&display=inline&height=480&name=image.png&originHeight=480&originWidth=1340&size=55546&status=done&width=1340)

`SELECT Carrier, avg(DepDelay > 10) * 100 AS c3 FROM ontime WHERE Year = 2018 GROUP BY Carrier ORDER BY Carrier` 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736271917-dadece08-87f4-4103-a298-170d115fe073.png#align=left&display=inline&height=503&name=image.png&originHeight=503&originWidth=1052&size=28083&status=done&width=1052)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736536023-b44eaf65-9397-4c7a-9574-43ecfede72ef.png#align=left&display=inline&height=254&name=image.png&originHeight=254&originWidth=297&size=6701&status=done&width=297)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736589364-dcb99f35-22ca-47d8-97a5-a98491f45bfe.png#align=left&display=inline&height=609&name=image.png&originHeight=609&originWidth=1347&size=108611&status=done&width=1347)

发布
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736715789-e21eea66-eb35-40a5-8061-100b79f2dac7.png#align=left&display=inline&height=312&name=image.png&originHeight=312&originWidth=1056&size=30412&status=done&width=1056)

创建仪表盘(Dashboard)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736796292-ac3b67dd-1d2f-41a2-a1db-a7e9456ba2cd.png#align=left&display=inline&height=557&name=image.png&originHeight=557&originWidth=1342&size=39921&status=done&width=1342)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736761829-08ae709d-eeda-46d7-a3b3-5d17e3eb38fe.png#align=left&display=inline&height=457&name=image.png&originHeight=457&originWidth=1021&size=43007&status=done&width=1021)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736856319-c2134885-f7fe-4468-9609-d1a1e155f220.png#align=left&display=inline&height=477&name=image.png&originHeight=477&originWidth=1359&size=90419&status=done&width=1359)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736915850-1639d58c-faf7-40f9-827b-5ad787fb52fb.png#align=left&display=inline&height=515&name=image.png&originHeight=515&originWidth=1354&size=109611&status=done&width=1354)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736878154-9f97437c-6b4f-419e-89bc-17f45f49947e.png#align=left&display=inline&height=273&name=image.png&originHeight=273&originWidth=551&size=17982&status=done&width=551)
分享后的 dashboard，在底下有个 redash 的 logo
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562736970500-ed1b5503-69c7-40e3-99d8-b5a7cdd58d82.png#align=left&display=inline&height=295&name=image.png&originHeight=295&originWidth=500&size=9731&status=done&width=500)

可以嵌入到已有系统里。

## 参考资料

- [我的博客](https://anjia0532.github.io/2019/07/08/redash/)
- [我的掘金](https://juejin.im/post/5d25b88cf265da1bc23f9ff3)
- [Knowledge Base](https://redash.io/help/)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊  [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
