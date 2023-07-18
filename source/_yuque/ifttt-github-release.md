---
title: 订阅github release
urlname: ifttt-github-release
date: '2019-01-29 12:32:00 +0800'
tags:
  - github
  - ifttt
categories: 其他
---

关注的 github repo 项目多了后，往往不能及时获取最新的 release 信息。本文整理了两种（官方自带+IFTTT）易于使用的订阅方式

<!-- more -->

## 官方自带 release 提醒

github 于 2018-11-28 日推出仅订阅 release 功能，喜大普奔啊，之前都是 IFTTT 或者自己写脚本轮或者订阅 rss

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1548733703278-4e1950fb-7799-49eb-8a83-9a0e03619544.png#align=left&display=inline&height=375&originHeight=375&originWidth=1023&size=78024&width=1023)

一旦有新的 release，就会发到 github 绑定的邮箱里了

详见  [Watching and unwatching releases for a repository](https://help.github.com/articles/watching-and-unwatching-releases-for-a-repository/)

## IFTTT

[IFTTT](https://ifttt.com)  是 `If This Then That`  ，简单来说就是如果(IF)发生了某事(This)那么就做啥(That).
用在此处就是如果关注的 github repo 发布了 release，那么就通知我(email 或者 weibo)

### 注册 IFTTT 账号

打开  [https://ifttt.com/join](https://ifttt.com/join)  填写邮箱和密码进行注册

### 创建 IFTTT 小程序

打开  [https://ifttt.com/create/](https://ifttt.com/create/)  选择 +this
选择 `RSS Feed` -> `New feed item` 
**Feed URL**  输入 `https://github.com/用户名/repo名/releases.atom`  比如我的一个 repo 的地址是  [https://github.com/anjia0532/elastalert-wechat-plugin/releases.atom](https://github.com/anjia0532/elastalert-wechat-plugin/releases.atom) 
点 `Create Trigger`  点 蓝色的 +that 部分
选择 `Email` -> `Send me an email` 
点击 `Create action`

一旦发布 releases 就会收到 email
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1548736026081-37c127af-53cf-4dec-8db3-df8706117d14.png#align=left&display=inline&height=546&originHeight=546&originWidth=783&size=60687&width=783)
可以通过  [https://ifttt.com/activity](https://ifttt.com/activity)  查看小程序运行情况

## Q&A

**Q: **有了 github 的 watch 了，为嘛还要折腾 IFTTT？
**A: **因为在 github 出来之前就用了 IFTTT 了，再一个，IFTTT 不光可以发 email，还可以发微博和 twitter，让你分分钟成为一个技术潮人。新鲜资讯早知道啊.
