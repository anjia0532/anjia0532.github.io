---
title: 074-Elastic Curator 支持ES8.x
urlname: curator-es-8-x
date: '2022-05-13 19:35:21 +0800'
tags:
  - es
  - elastic
  - elk
categories:
  - elk
---

> 这是坚持技术写作计划（含翻译）的第 74 篇，定个小目标 999，每周最少 2 篇。

ES 堪称版本帝, 截止到现在已经 ES 8.2.0 了. 但是悲催的是 Curator 目前只支持到 ES 7.x .

Curator 是一个 ES 工具集, 用于减少 ES 的运维工作量, 比如创建索引,设置索引参数,关闭索引,归档索引,删除索引,定时备份索引等等 . 支持的操作有 [https://www.elastic.co/guide/en/elasticsearch/client/curator/current/actions.html](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/actions.html)

其中 Curator 的部分功能,可以使用高版本的 ES 倡导的 ILM 功能 , 感兴趣,可以参考 [干货 | Elasticsearch 索引生命周期管理 ILM 实战指南](https://mp.weixin.qq.com/s/7VQd5sKt_PH56PFnCrUOHQ)

本文主要讲解如何让 Curator 支持 ES 8.x .

<!-- more -->

详见我的 Gist [https://gist.github.com/anjia0532/d8e779c97333d6fc0d045675e18e1224](https://gist.github.com/anjia0532/d8e779c97333d6fc0d045675e18e1224)

将下面代码保存成 `curator.patch`

```diff
Index: curator/_version.py
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/curator/_version.py b/curator/_version.py
--- a/curator/_version.py	(revision 191de8cd0c7c719de6d5fca1f9fbbc8adfb5c006)
+++ b/curator/_version.py	(revision caef1c9e29c67d534764ed25e4bf3f1ae70b6276)
@@ -1,2 +1,2 @@
 """Curator Version"""
-__version__ = '5.8.4'
+__version__ = '5.8.5'
Index: curator/defaults/settings.py
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/curator/defaults/settings.py b/curator/defaults/settings.py
--- a/curator/defaults/settings.py	(revision 191de8cd0c7c719de6d5fca1f9fbbc8adfb5c006)
+++ b/curator/defaults/settings.py	(revision caef1c9e29c67d534764ed25e4bf3f1ae70b6276)
@@ -6,7 +6,7 @@
 # Elasticsearch versions supported
 def version_max():
     """Return the maximum Elasticsearch version Curator supports"""
-    return (7, 99, 99)
+    return (8, 99, 99)
 def version_min():
     """Return the minimum Elasticsearch version Curator supports"""
     return (5, 0, 0)
Index: requirements.txt
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/requirements.txt b/requirements.txt
--- a/requirements.txt	(revision 191de8cd0c7c719de6d5fca1f9fbbc8adfb5c006)
+++ b/requirements.txt	(revision caef1c9e29c67d534764ed25e4bf3f1ae70b6276)
@@ -1,10 +1,10 @@
 voluptuous>=0.12.1
-elasticsearch>=7.14.0,<8.0.0
+elasticsearch>=7.14.0,<=9.0.0
 urllib3>=1.26.5,<2
 requests>=2.26.0
 boto3>=1.18.18
 requests_aws4auth>=1.1.1
-click>=7.0,<8.0
+click>=8.0.3,<9.0
 pyyaml>=5.4.1
 certifi>=2021.5.30
 six>=1.16.0
Index: setup.py
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/setup.py b/setup.py
--- a/setup.py	(revision 191de8cd0c7c719de6d5fca1f9fbbc8adfb5c006)
+++ b/setup.py	(revision caef1c9e29c67d534764ed25e4bf3f1ae70b6276)
@@ -22,12 +22,13 @@
     return VERSION

 def get_install_requires():
-    res = ['elasticsearch>=7.14.0,<8.0.0' ]
+    res = ['elasticsearch>=7.14.0,<=9.0.0' ]
+    res.append('voluptuous>=0.12.1')
     res.append('urllib3>=1.26.5,<2')
     res.append('requests>=2.26.0')
     res.append('boto3>=1.18.18')
     res.append('requests_aws4auth>=1.1.1')
-    res.append('click>=7.0,<8.0')
+    res.append('click>=8.0.3,<9.0')
     res.append('pyyaml==5.4.1')
     res.append('voluptuous>=0.12.1')
     res.append('certifi>=2021.5.30')
```

将下面代码保存成 `Dockerfile`

```dockerfile
FROM python:3.9.4-alpine3.13

# 国内加速器
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories && \
     pip install -i https://mirrors.ustc.edu.cn/pypi/web/simple pip -U && \
     pip config set global.index-url https://mirrors.ustc.edu.cn/pypi/web/simple

COPY curator.patch /tmp/curator.patch

# 如果下载 github 超时,可以换成 gitee 源 https://gitee.com/anjia/curator.git
RUN  apk --no-cache add --virtual .build-deps git && \
     git clone https://github.com/elastic/curator.git /tmp/curator && \
     cd /tmp/curator && \
     git checkout -b tags/v5.8.4 && \
     git apply --check /tmp/curator.patch && \
     git apply /tmp/curator.patch && \
     apk del .build-deps

RUN cd /tmp/curator && python setup.py install

# Thanks for @arslanbekov
RUN  rm -rf /tmp/curator
```

```shell
docker build . -t curator:5.8.5

# 或者也可以用我构建好的

docker pull anjia0532/curator:5.8.5
```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/05/13/curator-es-8-x/)
- [我的掘金](https://juejin.cn/post/7097613604978950175/)
- [Curator not compatible with elasticsearch 8.0.0 #1639](https://github.com/elastic/curator/issues/1639)
