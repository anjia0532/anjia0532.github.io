
---
title: 订阅github release
date: 2019-01-29 12:32:00 +0800
tags: [github,ifttt]
categories: 其他
---

关注的github repo项目多了后，往往不能及时获取最新的release信息。本文整理了两种（官方自带+IFTTT）易于使用的订阅方式

<!-- more -->
## 官方自带release提醒
github于2018-11-28日推出仅订阅release功能，喜大普奔啊，之前都是IFTTT或者自己写脚本轮或者订阅rss

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1548733703278-4e1950fb-7799-49eb-8a83-9a0e03619544.png#align=left&display=inline&height=375&linkTarget=_blank&name=image.png&originHeight=375&originWidth=1023&size=78024&width=1023)

一旦有新的release，就会发到github绑定的邮箱里了

详见 [Watching and unwatching releases for a repository](https://help.github.com/articles/watching-and-unwatching-releases-for-a-repository/)


## IFTTT
[IFTTT](https://ifttt.com) 是 `If This Then That`  ，简单来说就是如果(IF)发生了某事(This)那么就做啥(That).<br />用在此处就是如果关注的github repo 发布了release，那么就通知我(email或者weibo)

### 注册IFTTT账号
打开 [https://ifttt.com/join](https://ifttt.com/join)  填写邮箱和密码进行注册

### 创建IFTTT小程序
打开 [https://ifttt.com/create/](https://ifttt.com/create/) 选择 +this<br />选择 `RSS Feed` -> `New feed item` <br />**Feed URL** 输入 `https://github.com/用户名/repo名/releases.atom` 比如我的一个repo的地址是 [https://github.com/anjia0532/elastalert-wechat-plugin/releases.atom](https://github.com/anjia0532/elastalert-wechat-plugin/releases.atom) <br />点 `Create Trigger` 点 蓝色的 +that 部分<br />选择 `Email` -> `Send me an email` <br />点击 `Create action` 

一旦发布releases 就会收到 email<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1548736026081-37c127af-53cf-4dec-8db3-df8706117d14.png#align=left&display=inline&height=546&linkTarget=_blank&name=image.png&originHeight=546&originWidth=783&size=60687&width=783)<br />可以通过 [https://ifttt.com/activity](https://ifttt.com/activity) 查看小程序运行情况

## Q&A
**Q: **有了github的watch了，为嘛还要折腾IFTTT？<br />**A: **因为在github出来之前就用了IFTTT了，再一个，IFTTT不光可以发email，还可以发微博和twitter，让你分分钟成为一个技术潮人。新鲜资讯早知道啊.

