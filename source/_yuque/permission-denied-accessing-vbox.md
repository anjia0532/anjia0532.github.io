---
title: 043-解决vagrant访问virtualbox共享文件夹报无权限问题(Permission denied)
urlname: permission-denied-accessing-vbox
date: '2019-08-27 12:33:10 +0800'
tags:
  - 虚拟机
  - kvm
  - vagrant
  - virtualbox
  - python
categories:
  - python
---

> 这是坚持技术写作计划（含翻译）的第 43 篇，定个小目标 999，每周最少 2 篇。

本文是在 windows 进行 jumpserver 二开时，需要用到 vbox 的共享文件夹(宿主<->虚拟机双向读写)，vagrant 默认登录用户是 vagrant，而 vbox 的挂载文件夹是 vboxsf 组的，故而访问时会报 `Permission denied`  问题。

python+vagrant+virtualbox 系列文章



- [036-win10 搭建 python 的 linux 开发环境(pycharm+vagrant+virtualbox)](https://juejin.im/post/5d3a55ece51d454f71439dd2)
- [037-vagrant 启动(up)后自动同步文件(rsync-auto)](https://juejin.im/post/5d562b5e5188252d43756db8)
- [040-解决 Linux 使用 virtualbox 共享文件夹问题](https://juejin.im/post/5d5695056fb9a06afd6600f0)
- [042-解决 win10 VirtualBox 无法启动(VERR_NEM_VM_CREATE_FAILED)](https://juejin.im/post/5d63869a51882559c41612c6)
- [043-解决 vagrant 访问 virtualbox 共享文件夹报无权限问题(Permission denied)](https://juejin.im/post/5d6493d6e51d456206115a2c)[D)](https://juejin.im/post/5d63869a51882559c41612c6)

<!-- more -->

## 解决办法

```bash
# 登录虚拟机
$ vagrant ssh
# 将当前用户添加到vboxsf组内
$ sudo usermod -aG vboxsf $USER
# 以vboxsf组的身份重新登录 (注销，重启也可以)
$ newgrp - vboxsf
```

也可以参见我在  Stack Exchange 下   [Permission denied when accessing VirtualBox shared folder when member of the vboxsf group](https://superuser.com/a/1475587/1081025)  的回答

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2019/08/27/permission-denied-accessing-vbox/)
- [我的掘金](https://juejin.im/post/5d6493d6e51d456206115a2c)
- [Permission denied when accessing VirtualBox shared folder when member of the vboxsf group](https://superuser.com/a/1475587/1081025)
