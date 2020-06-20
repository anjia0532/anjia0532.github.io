---
title: 014-活该你爬虫被封之Scrapy Ip代理中间件
urlname: scrapy-proxy
date: 2019-04-02 20:41:00 +0800
tags: [python,scrapy,proxy]
categories: [爬,虫]
---

> 这是坚持技术写作计划（含翻译）的第 14 篇，定个小目标 999，每周最少 2 篇。

背景: 房租到期了。
需求: 找到便宜，交通便利的房源，了解当前租房行情，便于砍价。

在爬取 58，赶集，链家，安居客的数据时，被封是常事，基于此，fork 并修改了两个库。用于抓取免费代理 ip，用于支持爬取租房数据。

注意：租房网站的数据，大概率失真，仅做参考。

<!-- more -->

其中部分数据截图

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1554210833561-38730012-a2d6-4d00-aed7-5576b112e7c5.png#align=left&display=inline&height=740&name=image.png&originHeight=740&originWidth=1868&size=451298&status=done&width=1868)

本文只介绍 Scrapy 的 ip 代理中间件，不多讲如何爬取租房网站数据以及数据分析，后边可能会写。

## 获取代理 ip

如果有付费的代理 ip 更好，如果没有的话，可以用我构建的 docker 镜像

```bash
docker run -p8765:8765 -d anjia0532/ipproxy-dockerfile
```

稍等 2-5 分钟，访问   http://${docker ip}:8765/ ,如果有值，则抓取代理 ip 成功。

## scrapy-proxies-tool

### 安装

```bash
pip install scrapy-proxies-tool
```

### 配置

修改 Scrapy settings.py，源[repo ](https://github.com/aivarsk/scrapy-proxies)只支持从文件读取代理 ip

```python
# Retry many times since proxies often fail
RETRY_TIMES = 10
# Retry on most error codes since proxies fail for different reasons
RETRY_HTTP_CODES = [500, 503, 504, 400, 403, 404, 408]

DOWNLOADER_MIDDLEWARES = {
  'scrapy.downloadermiddlewares.retry.RetryMiddleware': 90,
  'scrapy_proxies.RandomProxy': 100,
  'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
}

PROXY_SETTINGS = {
  # Proxy list containing entries like
  # http://host1:port
  # http://username:password@host2:port
  # http://host3:port
  # ...
  # if PROXY_SETTINGS[from_proxies_server] = True , proxy_list is server address (ref https://github.com/qiyeboy/IPProxyPool and https://github.com/awolfly9/IPProxyTool )
  # Only support http(ref https://github.com/qiyeboy/IPProxyPool#%E5%8F%82%E6%95%B0)
  # list : ['http://localhost:8765?protocol=0'],
  'list':['/path/to/proxy/list.txt'],

  # disable proxy settings and  use real ip when all proxies are unusable
  'use_real_when_empty':False,
  'from_proxies_server':False,

  # If proxy mode is 2 uncomment this sentence :
  # 'custom_proxy': "http://host1:port",

  # Proxy mode
  # 0 = Every requests have different proxy
  # 1 = Take only one proxy from the list and assign it to every requests
  # 2 = Put a custom proxy to use in the settings
  'mode':0
}
```

可以通过爬取  [http://myip.ipip.net/](http://myip.ipip.net/)  来判断代理 ip 是否生效。

## 参考资料

- [https://github.com/anjia0532/IPProxyPool](https://github.com/anjia0532/IPProxyPool)
- [https://hub.docker.com/r/anjia0532/ipproxy-dockerfile](https://hub.docker.com/r/anjia0532/ipproxy-dockerfile)
- [https://github.com/aivarsk/scrapy-proxies](https://github.com/aivarsk/scrapy-proxies)
- [https://github.com/anjia0532/scrapy-proxies](https://github.com/anjia0532/scrapy-proxies)
