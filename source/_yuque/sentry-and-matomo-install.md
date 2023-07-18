---
title: 030-前端错误日志上报及网站统计(sentry+matomo)
urlname: sentry-and-matomo-install
date: '2019-07-07 15:00:00 +0800'
tags:
  - sentry
  - 运维
  - 日志
  - 前端
  - vuejs
  - matomo
  - 网站分析
categories: 运维
---

> 这是坚持技术写作计划（含翻译）的第 30 篇，定个小目标 999，每周最少 2 篇。

本文配合 rancher1.6(手头一个测试集群没升级到最新的 2.x)讲解如何搭建并配置日志错误上报框架[Sentry](https://sentry.io)及网站统计分析框架[matomo](https://matomo.org/)  的搭建及接入 vue(本文以[iview-admin](https://github.com/iview/iview-admin)为例)项目。

## 背景简述

- sentry 项目运行过程中，难免出现 bug，前端不像后端可以很方便的采集项目日志(比如 log4j + elk)，导致每次出问题后还原车祸现场费时费力。另外现在随着 vue 等兴起，构建项目时打成 min.js，无疑又加大了前端定位问题的难度。而 sentry 是一款专注于错误采集的框架，支持常见的主流语言![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562489039361-12bbbf21-c134-424c-a4ef-24a8d5ed536b.png#align=left&display=inline&height=544&originHeight=544&originWidth=1779&size=112141&status=done&width=1779)采集，聚合，并推送错误信息。注意，sentry 并不是日志平台(e.g. log4j + elk)，也不是监控平台，sentry 专注于项目中的 Error 信息的采集，聚合，报警。
- matomo 前身 Pwiki，是一款开源的 web 网站分析利器。类似于 Google Analytics。具体的特性，参见  [Premium Web Analytics](https://matomo.org/feature-overview/) ，比如绘制页面热力图，录制会话，访问漏斗，A/B Test 等(这几样都是收费插件功能)。

> 注意：本文假设你已经有 rancher1.6 的环境

<!-- more -->

## 安装

### matomo

#### rancher 创建 matomo

在 rancher 主机上

```bash
## 创建必要文件夹
mkdir -p /data/matomo/{config,logs,php,maxmind}/

## 安装maxmind ip数据库
wget -P /tmp/ https://github.com/maxmind/geoipupdate/releases/download/v4.0.3/geoipupdate_4.0.3_linux_amd64.deb
dpkg -i /tmp/geoipupdate_4.0.3_linux_amd64.deb
mv /etc/GeoIP.conf{,.bak}
cat << EOF | sudo tee -a /etc/GeoIP.conf
AccountID 0
LicenseKey 000000000000
EditionIDs GeoLite2-Country GeoLite2-City GeoLite2-ASN
DatabaseDirectory /data/matomo/maxmind
EOF

## 下载最新maxmind数据库
geoipupdate
ls -lah /data/matomo/maxmind/
总用量 67M
drwxr-xr-x 2 root root 4.0K 7月   7 17:25 .
drwxr-xr-x 6 root root 4.0K 7月   7 17:23 ..
-rw------- 1 root root    0 7月   7 17:25 .geoipupdate.lock
-rw-r--r-- 1 root root 6.3M 7月   7 17:25 GeoLite2-ASN.mmdb
-rw-r--r-- 1 root root  57M 7月   7 17:25 GeoLite2-City.mmdb
-rw-r--r-- 1 root root 3.7M 7月   7 17:25 GeoLite2-Country.mmdb

## 定时更新ip数据库
cat << EOF | sudo tee -a /etc/cron.d/geoipupdate
50 2 * * 4 root /usr/bin/geoipupdate
EOF

## 设置php.ini
cat << EOF | sudo tee -a /data/matomo/php/php.ini
upload_max_filesize = 128M
post_max_size = 128M
max_execution_time = 200
memory_limit = 256M
EOF
```

docker-compose.yaml

```yaml
version: "2"
services:
  matomo:
    image: matomo:latest
    stdin_open: true
    volumes:
      - /data/matomo/config:/var/www/html/config
      - /data/matomo/logs:/var/www/html/logs
      - /data/matomo/php/php.ini:/usr/local/etc/php/php.ini
      - /data/matomo/maxmind/GeoLite2-City.mmdb:/var/www/html/misc/GeoLite2-City.mmdb
      - /data/matomo/maxmind/GeoLite2-Country.mmdb:/var/www/html/misc/GeoLite2-Country.mmdb
      - /data/matomo/maxmind/GeoLite2-ASN.mmdb:/var/www/html/misc/GeoLite2-ASN.mmdb
    tty: true
    ports:
      - 80:80/tcp
      - 443:443/tcp
```

rancher-compose.yaml

```yaml
version: "2"
services:
  matomo:
    scale: 1
    start_on_create: true
    health_check:
      response_timeout: 2000
      healthy_threshold: 2
      port: 80
      unhealthy_threshold: 3
      initializing_timeout: 60000
      interval: 2000
      strategy: recreate
      reinitializing_timeout: 60000
```

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562492855584-f8bc4414-c57b-4780-94ee-60f0c42258fb.png#align=left&display=inline&height=588&originHeight=588&originWidth=1884&size=72142&status=done&width=1884)

#### 配置 matomo

选择中文
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562474861136-8a51035c-9f3d-45c9-b560-17a8d251f21b.png#align=left&display=inline&height=508&originHeight=508&originWidth=606&size=39934&status=done&width=606)
系统检查
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562474876102-d3669793-8f7c-4884-a7c8-57c6892b0a63.png#align=left&display=inline&height=602&originHeight=602&originWidth=1361&size=59313&status=done&width=1361)
配置数据库
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562475009264-1585cc06-9880-4eb3-9908-70e78cf9012e.png#align=left&display=inline&height=830&originHeight=830&originWidth=823&size=53486&status=done&width=823)
自动建表完成
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562475051907-56c50bd2-4e8a-4696-9e99-929d43bd60c8.png#align=left&display=inline&height=520&originHeight=520&originWidth=1348&size=46284&status=done&width=1348)
创建管理员用户(忘记截图了)
设置分析网站(可以随便创建，后边再进行修改),注意跟进实际情况修改时区
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562475212464-ed37c762-0452-4ae2-9976-c4a90d8eadda.png#align=left&display=inline&height=711&originHeight=711&originWidth=936&size=51354&status=done&width=936)
复制跟踪代码
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562475254242-9a8baee0-dc7e-47ff-a913-cca4bcecf018.png#align=left&display=inline&height=844&originHeight=844&originWidth=1396&size=113598&status=done&width=1396)
配置 matomo
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562475288189-d137a11a-cb01-4311-955e-d42788988f21.png#align=left&display=inline&height=843&originHeight=843&originWidth=1172&size=109853&status=done&width=1172)
登陆
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562475319066-c88fd562-3712-4022-a040-f64b1cbbafbe.png#align=left&display=inline&height=668&originHeight=668&originWidth=798&size=26965&status=done&width=798)

### Sentry

#### rancher 创建 Sentry

docker-compose.yml

```yaml
version: "2"
services:
  cron:
    image: sentry:9
    environment:
      SENTRY_MEMCACHED_HOST: memcached
      SENTRY_REDIS_HOST: redis
      SENTRY_POSTGRES_HOST: postgres
      SENTRY_EMAIL_HOST: smtp
      SENTRY_SECRET_KEY: SENTRY_SECRET_KEY_XXX
    stdin_open: true
    volumes:
      - /data/sentry-data:/var/lib/sentry/files
      - /data/sentry-data/config.yml:/etc/sentry/config.yml
    tty: true
    command:
      - run
      - cron
  memcached:
    image: memcached:1.5-alpine
  web:
    image: sentry:9
    environment:
      SENTRY_MEMCACHED_HOST: memcached
      SENTRY_REDIS_HOST: redis
      SENTRY_POSTGRES_HOST: postgres
      SENTRY_EMAIL_HOST: smtp
      SENTRY_SECRET_KEY: SENTRY_SECRET_KEY_XXX
    stdin_open: true
    volumes:
      - /data/sentry-data:/var/lib/sentry/files
      - /data/sentry-data/config.yml:/etc/sentry/config.yml
    tty: true
    ports:
      - 9000:9000/tcp
  worker:
    image: sentry:9
    environment:
      SENTRY_MEMCACHED_HOST: memcached
      SENTRY_REDIS_HOST: redis
      SENTRY_POSTGRES_HOST: postgres
      SENTRY_EMAIL_HOST: smtp
      SENTRY_SECRET_KEY: SENTRY_SECRET_KEY_XXX
    stdin_open: true
    volumes:
      - /data/sentry-data:/var/lib/sentry/files
      - /data/sentry-data/config.yml:/etc/sentry/config.yml
    tty: true
    command:
      - run
      - worker
  redis:
    image: redis:3.2-alpine
  postgres:
    restart: unless-stopped
    image: postgres:9.5
    volumes:
      - /data/postgresql/data:/var/lib/postgresql/data
    ports:
      - 5432:5432/tcp
```

> 注意把   SENTRY_SECRET_KEY 换成 sentry 的实际 secret key

rancher-compose.yml

```yaml
version: "2"
services:
  cron:
    scale: 1
    start_on_create: true
  memcached:
    scale: 1
    start_on_create: true
  web:
    scale: 1
    start_on_create: true
    health_check:
      response_timeout: 2000
      healthy_threshold: 2
      port: 9000
      unhealthy_threshold: 3
      initializing_timeout: 60000
      interval: 2000
      strategy: recreate
      reinitializing_timeout: 60000
  worker:
    scale: 1
    start_on_create: true
  redis:
    scale: 1
    start_on_create: true
  postgres:
    scale: 1
    start_on_create: true
    health_check:
      response_timeout: 2000
      healthy_threshold: 2
      port: 5432
      unhealthy_threshold: 3
      initializing_timeout: 60000
      interval: 2000
      strategy: recreate
      reinitializing_timeout: 60000
```

先将 docker-compose.yml 保存到服务器上，用来初始化 db 和创建账号

```bash
docker-compose run --rm web upgrade
Would you like to create a user account now? [Y/n]: y
Email: anjia0532@gmail.com
Password:
Repeat for confirmation:
Should this user be a superuser? [y/N]: y
## 直到输出
Migrated:
 - sentry
 - sentry.nodestore
 - sentry.search
 - social_auth
 - sentry.tagstore
 - sentry_plugins.hipchat_ac
 - sentry_plugins.jira_ac
Creating missing DSNs
Correcting Group.num_comments counter
## 并退出
```

#### 配置 Sentry

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562504762029-c863b8c3-4b57-4260-b370-e538cce436e7.png#align=left&display=inline&height=503&originHeight=503&originWidth=716&size=44203&status=done&width=716)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562504816401-106637ac-f528-4985-bcd1-f5649f00cc18.png#align=left&display=inline&height=712&originHeight=712&originWidth=656&size=148457&status=done&width=656)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562504900957-00a0a900-8276-47cf-b33a-52fe87cdd130.png#align=left&display=inline&height=804&originHeight=804&originWidth=1501&size=130999&status=done&width=1501)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562505169347-a858daf1-bd4d-4101-97d5-aa071564a8db.png#align=left&display=inline&height=222&originHeight=222&originWidth=258&size=5513&status=done&width=258)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562505136746-68d7b0e7-932e-4aef-be34-12da8cc9bbf7.png#align=left&display=inline&height=575&originHeight=575&originWidth=1029&size=69263&status=done&width=1029)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562505216529-4a3b7152-0b37-4014-a9e2-266cff0b72d7.png#align=left&display=inline&height=620&originHeight=620&originWidth=1281&size=95572&status=done&width=1281)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562505244285-e97409c9-7565-4b53-b4da-d669bf2db3ad.png#align=left&display=inline&height=894&originHeight=894&originWidth=1272&size=139045&status=done&width=1272)

## 配置 vue

本文以 iview-admin 为例

```bash
git clone https://gitee.com/anjia/iview-admin.git
cd iview-admin
```

### sentry

注意，网上很多文档，以讹传讹的使用过时的工具，raven-js .从 5.x 后官方建议使用@sentry/browser 和@sentry/integrations

```bash
npm install --save @sentry/integrations
npm install --save @sentry/browser
```

修改 `iview-admin\src\main.js`

```javascript
// The Vue build version to load with the `import` command
// (runtime-only or standalone) has been set in webpack.base.conf with an alias.
import Vue from "vue";
import App from "./App";
import router from "./router";
import store from "./store";
import iView from "iview";
import i18n from "@/locale";
import config from "@/config";
import importDirective from "@/directive";
import { directive as clickOutside } from "v-click-outside-x";
import installPlugin from "@/plugin";
import "./index.less";
import "@/assets/icons/iconfont.css";
import TreeTable from "tree-table-vue";
import VOrgTree from "v-org-tree";
import "v-org-tree/dist/v-org-tree.css";
import * as Sentry from "@sentry/browser";
import * as Integrations from "@sentry/integrations";
// 实际打包时应该不引入mock
/* eslint-disable */
if (process.env.NODE_ENV !== "production") require("@/mock");

Vue.use(iView, {
  i18n: (key, value) => i18n.t(key, value),
});
Vue.use(TreeTable);
Vue.use(VOrgTree);
/**
 * @description 注册admin内置插件
 */
installPlugin(Vue);
/**
 * @description 生产环境关掉提示
 */
Vue.config.productionTip = false;
/**
 * @description 全局注册应用配置
 */
Vue.prototype.$config = config;
/**
 * 注册指令
 */
importDirective(Vue);
Vue.directive("clickOutside", clickOutside);

/* eslint-disable no-new */
new Vue({
  el: "#app",
  router,
  i18n,
  store,
  render: (h) => h(App),
});

Sentry.init({
  dsn: "https://xxx@xxx.xxx.com/xxx",
  integrations: [
    new Integrations.Vue({
      Vue,
      attachProps: true,
    }),
  ],
});
```

```bash
npm install
npm run dev
```

打开  [http://localhost:8080/error_store/error_store_page](http://localhost:8080/error_store/error_store_page)
分别点击两个按钮，模拟出错
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562507067736-b37abeba-06bb-48cf-9421-7229b0250070.png#align=left&display=inline&height=797&originHeight=797&originWidth=913&size=108137&status=done&width=913)
打开 sentry 发现已经有错误上报了，并且对错误进行聚合。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562507097119-284acf63-b1df-4004-9027-512dad7b3af3.png#align=left&display=inline&height=464&originHeight=464&originWidth=1783&size=102500&status=done&width=1783)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562507211900-c2fcf0a4-2265-43e5-8ed4-d41ce849b37c.png#align=left&display=inline&height=927&originHeight=927&originWidth=1533&size=181472&status=done&width=1533)点开查看详细内容。

如果需要生成 source-map ,可以参考官方文档  [https://docs.sentry.io/platforms/javascript/sourcemaps/](https://docs.sentry.io/platforms/javascript/sourcemaps/)

### matomo

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562507978926-cd3d0612-a601-4294-bd27-afe1aa95e76f.png#align=left&display=inline&height=615&originHeight=615&originWidth=1843&size=53616&status=done&width=1843)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562508055404-00bd3ea6-b9d9-4911-9d73-84e9b40d3499.png#align=left&display=inline&height=277&originHeight=277&originWidth=1084&size=18426&status=done&width=1084)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562508079973-2f10991b-1f19-4403-bc6f-3d62694e0abd.png#align=left&display=inline&height=559&originHeight=559&originWidth=633&size=35279&status=done&width=633)

```bash
npm install --save vue-matomo
```

修改  `iview-admin\src\main.js`

```javascript
// The Vue build version to load with the `import` command
// (runtime-only or standalone) has been set in webpack.base.conf with an alias.
import Vue from "vue";
import App from "./App";
import router from "./router";
import store from "./store";
import iView from "iview";
import i18n from "@/locale";
import config from "@/config";
import importDirective from "@/directive";
import { directive as clickOutside } from "v-click-outside-x";
import installPlugin from "@/plugin";
import "./index.less";
import "@/assets/icons/iconfont.css";
import TreeTable from "tree-table-vue";
import VOrgTree from "v-org-tree";
import "v-org-tree/dist/v-org-tree.css";
import * as Sentry from "@sentry/browser";
import * as Integrations from "@sentry/integrations";
import VueMatomo from "vue-matomo";

// 实际打包时应该不引入mock
/* eslint-disable */
if (process.env.NODE_ENV !== "production") require("@/mock");

Vue.use(iView, {
  i18n: (key, value) => i18n.t(key, value),
});
Vue.use(TreeTable);
Vue.use(VOrgTree);
/**
 * @description 注册admin内置插件
 */
installPlugin(Vue);
/**
 * @description 生产环境关掉提示
 */
Vue.config.productionTip = false;
/**
 * @description 全局注册应用配置
 */
Vue.prototype.$config = config;
/**
 * 注册指令
 */
importDirective(Vue);
Vue.directive("clickOutside", clickOutside);

/* eslint-disable no-new */
new Vue({
  el: "#app",
  router,
  i18n,
  store,
  render: (h) => h(App),
});

Sentry.init({
  dsn: "https://xxx@xxx.xxx.com/xxx",
  integrations: [
    new Integrations.Vue({
      Vue,
      attachProps: true,
    }),
  ],
});
Vue.use(VueMatomo, {
  // Configure your matomo server and site by providing
  host: "//xxxx.xxxx.com/",
  siteId: xx,

  // Changes the default .js and .php endpoint's filename
  // Default: 'piwik'
  trackerFileName: "matomo.js",

  // Overrides the autogenerated tracker endpoint entirely
  // Default: undefined
  trackerUrl: "//xxxx.xxxx.com/matomo.php",

  // Enables automatically registering pageviews on the router
  router: router,

  // Enables link tracking on regular links. Note that this won't
  // work for routing links (ie. internal Vue router links)
  // Default: true
  enableLinkTracking: true,

  // Require consent before sending tracking information to matomo
  // Default: false
  requireConsent: false,

  // Whether to track the initial page view
  // Default: true
  trackInitialView: true,

  // Whether or not to log debug information
  // Default: false
  debug: false,
});

// or
window._paq.push;

// or through
window.Piwik.getTracker;
```

打开 http://localhost:8080, 随便访问几个菜单,然后打开 matomo
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562508801909-af15cc0f-180a-4cfc-86ec-768bfd892d67.png#align=left&display=inline&height=515&originHeight=515&originWidth=880&size=45838&status=done&width=880)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562508853543-f7592d90-ff5e-4f58-974a-e1f37b230e2c.png#align=left&display=inline&height=727&originHeight=727&originWidth=1416&size=69029&status=done&width=1416)
路由已经有数据了
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1562508892758-378acda5-d8ee-44a1-81f9-7ac7933da610.png#align=left&display=inline&height=749&originHeight=749&originWidth=1553&size=122920&status=done&width=1553)
并且将用户的常规数据聚合起来

## 更多

其实本文只是 sentry 和 matomo 简单介绍
更深入的使用，比如 sentry，推送邮件，文中一带而过的 sourcemap，单点登录(集成内部的权限认证)，自定义上报内容(将错误与用户 id 关联起来)，,敏感数据脱敏等
比如 matomo, 每日发送分析报表，增加 kafka 插件，进行更深层次的挖掘，自定义上报内容(购物车等),大数据量情况下的优化，优化用户设备指纹，使用了 nginx 等反代软件后，如何正确识别真实 ip，热力图，A/B test,漏斗图等

## 参考资料

- [我的个人博客](https://anjia0532.github.io/2019/07/07/sentry-and-matomo-install)
- [我的掘金](https://juejin.im/post/5d22013de51d45775a700395)
- [matomo 官网](https://matomo.org/)
- [INSTALLING MATOMO](https://matomo.org/docs/installation/)
- [Sentry 官网](https://sentry.io)
- [Installation with Docker](https://docs.sentry.io/server/installation/docker/)
- [Automatic Updates for GeoIP2 and GeoIP Legacy Databases](https://dev.maxmind.com/geoip/geoipupdate/)
- [vue 中如何使用 sentry 对错误日志进行监控](https://segmentfault.com/a/1190000016309667)
