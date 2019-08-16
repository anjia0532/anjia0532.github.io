
---

title: 029-解决sentry禁用qq邮箱问题

date: 2019-07-01 20:00:00 +0800

tags: [sentry,运维,日志]

categories: 运维

---
> 这是坚持技术写作计划（含翻译）的第29篇，定个小目标999，每周最少2篇。


Sentry是一款错误日志采集、聚合框架。有Saas版，也可以本地部署。部署可以参考官网或者我之前写的 [30-前端错误日志上报及网站统计(sentry+matomo)](https://anjia0532.github.io/2019/07/07/sentry-and-matomo-install/)

本文主要讲解Sentry默认禁用qq邮箱的排查思路以及如何解决。

添加和自行注册qq邮箱都报无效邮箱。<br />![](https://cdn.nlark.com/yuque/0/2019/png/226273/1561966878110-16806fe5-58ff-4d1d-8287-933328fd8819.png#align=left&display=inline&height=240&originHeight=456&originWidth=1418&size=0&status=done&width=746)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561967139251-c939294d-9ed6-4301-8c4b-c51719989847.png#align=left&display=inline&height=382&name=image.png&originHeight=382&originWidth=633&size=27110&status=done&width=633)

但是QQ邮箱，烂归烂，在国内存量还是挺大的。

<!-- more -->

<a name="DyCB3"></a>
## 排查思路
F12大法，看到错误信息是从服务端返回的。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561967659211-59f1fb29-96aa-4625-bead-5d07788aa142.png#align=left&display=inline&height=525&name=image.png&originHeight=525&originWidth=1144&size=76127&status=done&width=1144)

拿到 错误提示 `Enter a valid email address.` 去github搜，<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561968321654-bdbf1381-1e9c-43aa-b7f8-0c612ce52933.png#align=left&display=inline&height=806&name=image.png&originHeight=806&originWidth=822&size=89255&status=done&width=822)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561968360680-7a687c4f-93d4-4ea6-97c9-ecad24a46ed8.png#align=left&display=inline&height=159&name=image.png&originHeight=159&originWidth=426&size=16407&status=done&width=426)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561968410999-fdcfeef4-7f04-461e-b3a6-a697a83ae7e2.png#align=left&display=inline&height=109&name=image.png&originHeight=109&originWidth=596&size=8716&status=done&width=596)<br />拿到 `INVALID_EMAIL_ADDRESS_PATTERN` 再次搜索<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561968483952-2f2b929e-e20a-4cff-999c-8a8f877f1f5a.png#align=left&display=inline&height=514&name=image.png&originHeight=514&originWidth=1080&size=60182&status=done&width=1080)<br />居然是硬编码到代码里的。发现有两个相关的issues.

通过 [getsentry/sentry How to custom INVALID_EMAIL_ADDRESS_PATTERN? #13541](https://github.com/getsentry/sentry/issues/13541) 了解到，官方发现，qq.com 是有很多滥用行为，所以直接硬编码拉黑 [捂脸] .

<a name="0Izxl"></a>
## 解决
解决的办法也简单，如果是本地运行的，修改 `/usr/local/lib/python2.7/site-packages/sentry/conf/server.py` ,如果是sass版的，换个邮箱。

注意，如果是 docker 运行的， `docker exec -it sentry /bin/sh`  -> `sed -i "s/qq/xx/g" /usr/local/lib/python2.7/site-packages/sentry/conf/server.py` ,重新拉取镜像时，又会变回 `qq.com` 

可以 将  `/usr/local/lib/python2.7/site-packages/sentry/conf/server.py`  挂载到宿主机 `docker run -v /opt/sentry/server.py:/usr/local/lib/python2.7/site-packages/sentry/conf/server.py ....` 

<a name="oz04Q"></a>
## 参考资料

- [我的个人博客](https://anjia0532.github.io/2019/07/07/sentry-and-matomo-install/)
- [我的掘金](https://juejin.im/post/5d22029b6fb9a07ecd3d7f25)
- [getsentry/sentry How to custom INVALID_EMAIL_ADDRESS_PATTERN? #13541](https://github.com/getsentry/sentry/issues/13541)

