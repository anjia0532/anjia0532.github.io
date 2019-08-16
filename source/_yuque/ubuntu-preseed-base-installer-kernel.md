
---

title: 039-解决ubuntu使用preseed装机 base-installer/kernel/failed-package-install 问题

date: 2019-08-16 18:30:07 +0800

tags: [pxe,dhcp,cloudboot,ubuntu,preseed]

categories: 运维

---

> 这是坚持技术写作计划（含翻译）的第39篇，定个小目标999，每周最少2篇。


本文主要介绍在使用pressed无人装机安装ubuntu时，偶尔出现<br />![](https://cdn.nlark.com/yuque/0/2019/png/226273/1565929377804-60abd47c-6890-4bca-b20b-7132192158e1.png#align=left&display=inline&height=595&originHeight=595&originWidth=801&size=0&status=done&width=801)

```bash
Unexpected error; command not executed: 'sh -c debconf-apt-progress --no-progress --logstederr -- apt-get -q -y --no-remove install busybox-initramfs'
base-installer: error: exiting on error base-installer/kernel/failed-package-install
```
的解决方案。

之前写的几篇无人装机的文章(有基于cobbler和cloudboot的)

- [010-cloudboot批量安装rancheros](https://juejin.im/post/5c84f9d8f265da2de33f5936)
- [007-Cobbler批量自动化部署Windows10和Server 2019及激活](https://juejin.im/post/5c748b2af265da2d9262ed0f)
- [006-Cobbler批量自动化部署CentOS/Ubuntu/Windows](https://juejin.im/post/5c748ae2f265da2d84108d71)

<!-- more -->

<a name="2ncpC"></a>
## 排查过程
首先，点击继续，返回上一层页面，选择shell, `cat /var/log/syslog` 找到报错信息 `base-installer: error: exiting on error base-installer/kernel/failed-package-install` <br />在百度和google搜索后，找到跟我类似的问题 [XenServer安装ubuntu16.04遇到的错误](https://imaojia.com/blog/questions/error-install-ubuntu-16-04-on-xenserver/) 

但是使用作者的方式处理一遍后，没啥效果，但是阴差阳错的get了 `ctrl+alt+f4` (参考 [Reverting from Ctrl - Alt - F1](https://askubuntu.com/a/157621) )

能看安装日志就简单了，排查就行了，发现是安装ubuntu security时，国外ip被ban了，换成国内源即可。

修改preseed

```bash
d-i apt-setup/services-select multiselect security
d-i apt-setup/security_host string mirrors.aliyun.com
d-i apt-setup/security_path string /ubuntu
```

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。<br />长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

<a name="35808e79"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/08/16/ubuntu-preseed-base-installer-kernel)
- [我的掘金](https://juejin.im/post/5d5638f45188255d51425ced)
- [XenServer安装ubuntu16.04遇到的错误](https://imaojia.com/blog/questions/error-install-ubuntu-16-04-on-xenserver/)
- [Reverting from Ctrl - Alt - F1](https://askubuntu.com/a/157621)
- [Headless Installation: Unable to install busybox-initramfs](https://askubuntu.com/questions/949826/headless-installation-unable-to-install-busybox-initramfs)
- [How do I configure a preseed to skip the language support question?](https://askubuntu.com/questions/129651/how-do-i-configure-a-preseed-to-skip-the-language-support-question/349841#349841)


