
---

title: 043-解决vagrant访问virtualbox共享文件夹报无权限问题(Permission denied)

date: 2019-08-27 12:33:10 +0800

tags: [虚拟机,kvm,vagrant,virtualbox,python]

categories: python

---

> 这是坚持技术写作计划（含翻译）的第43篇，定个小目标999，每周最少2篇。


本文是在windows进行jumpserver二开时，需要用到vbox的共享文件夹(宿主<->虚拟机双向读写)，vagrant默认登录用户是vagrant，而vbox的挂载文件夹是vboxsf组的，故而访问时会报 `Permission denied` 问题。

python+vagrant+virtualbox系列文章<br /> 

- [036-win10搭建python的linux开发环境(pycharm+vagrant+virtualbox)](https://juejin.im/post/5d3a55ece51d454f71439dd2) 
- [037-vagrant启动(up)后自动同步文件(rsync-auto)](https://juejin.im/post/5d562b5e5188252d43756db8) 
- [040-解决Linux使用virtualbox共享文件夹问题](https://juejin.im/post/5d5695056fb9a06afd6600f0)
- [042-解决win10 VirtualBox无法启动(VERR_NEM_VM_CREATE_FAILED)](https://juejin.im/post/5d63869a51882559c41612c6)
- [043-解决vagrant访问virtualbox共享文件夹报无权限问题(Permission denied)](https://juejin.im/post/5d6493d6e51d456206115a2c)[D)](https://juejin.im/post/5d63869a51882559c41612c6)

<!-- more -->
<a name="uaQa2"></a>
## 解决办法

```bash
# 登录虚拟机
$ vagrant ssh
# 将当前用户添加到vboxsf组内
$ sudo usermod -aG vboxsf $USER 
# 以vboxsf组的身份重新登录 (注销，重启也可以)
$ newgrp - vboxsf
```
也可以参见我在 Stack Exchange下  [Permission denied when accessing VirtualBox shared folder when member of the vboxsf group](https://superuser.com/a/1475587/1081025) 的回答
<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。<br />长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

<a name="35808e79"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/08/27/permission-denied-accessing-vbox/)
- [我的掘金](https://juejin.im/post/5d6493d6e51d456206115a2c)
- [Permission denied when accessing VirtualBox shared folder when member of the vboxsf group](https://superuser.com/a/1475587/1081025)

