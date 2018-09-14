---
title: (2018年9月可用)官网下载免费版xshell5
date: 2018-09-14 09:18:40
tags: [linux,xshell,xshell5]
---



xshell 是一个强大的终端模拟软件，类似的还有很多，比如 mobaxterm ,putty, Securecrt(本文不过多介绍，感兴趣自行百度)



回到xshell, 自从手贱升级到xshell6以后,发现只能支持4个tab页，尴尬症都犯了，想降回xshell5，发现官方已经关闭5的下载通道了，难道你敢从百度等下载站随便下载xshell么？反正我不敢，经过一通折腾，终于成功找到官方下载入口了。

<!--more-->



根据 [v2ex](https://www.v2ex.com/t/460182), 其中 [chanssl](https://www.v2ex.com/t/460182) 指出

Xshell: <http://www.netsarang.com/download/down_live.html?productcode=2&majorversion=5>

Xftp: <http://www.netsarang.com/download/down_live.html?productcode=3&majorversion=5>

的办法，发现下载的是评估版的，也就是30天可用。而且最蛋疼的是不能输入注册码，所以网上找的注册码都GG了。



途中又尝试了`https://www.netsarang.com/download/down_form.html?code=622`改成`https://www.netsarang.com/download/down_form.html?code=522`发现只能输入`Product key` ，`Evaluation user / Home & School user` 压根就被删除了



难道只能下载免费的6或者换别的终端软件？



不死心的又尝试了一下F12大法，打开 https://www.netsarang.com/download/down_form.html?code=622

在 `Console`执行 `document.getElementById("code").value=522`

![](http://ww1.sinaimg.cn/large/afaffa71ly1fv8vsmrgpvj20f40drjrx.jpg)

然后打开邮箱，查看有没有收到下载链接

![](http://ww1.sinaimg.cn/large/afaffa71ly1fv8vu17nywj21380l7gnf.jpg)

这属于程序漏洞，所以随着本文的传播，很可能被堵上。(截止2018-09-14本方法仍可用)

所以，一旦下载成功后，千万千万记得在云盘存一份。防止后期系统重装时，本方法不可用了，又信不过网上提供的下载链接，就抓瞎了。



此处贴一下`Xshell-5.0.1339p.exe`校验码

```
大小：	33, 012, 688 字节
MD5：	AB1A4AFB4894B71A3DC4DE84A84E7126
SHA1：	D2DA24229554139AEF8D21F737D6F78F7BEF7A7F
CRC32：305847D5

```

至于为啥跟 https://www.netsarang.com/download/down_live.html?productcode=2&majorversion=5 的校验码(`MD5: 6a2aef6ac7a502f31607524a13659a81`)不一致，因为 https://www.netsarang.com/download/down_live.html?productcode=2&majorversion=5  是评估版，并且文件名是`Xshell-5.0.1339.exe`

另：出于对看客服务器的安全考虑，本文不会提供二进制下载。



博客 [https://anjia0532.github.io/2018/09/14/xshell5/](https://anjia0532.github.io/2018/09/14/xshell5/)
掘金 [https://juejin.im/post/5b9b271be51d450e7e513d88](https://juejin.im/post/5b9b271be51d450e7e513d88)