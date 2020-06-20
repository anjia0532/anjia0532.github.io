
---

title: 037-vagrant启动(up)后自动同步文件(rsync-auto)

date: 2019-07-29 08:37:45 +0800

tags: []

---
date: 2019-08-14 12:30:00
tags: [虚拟机,kvm,vagrant,virtualbox,python]
categories: python
---

> 这是坚持技术写作计划（含翻译）的第37篇，定个小目标999，每周最少2篇。


本文介绍两种vagrant up后自动同步文件(rsync) 分别基于 `sync` 和 `nfs` (如果不设置的话，需要再起一个终端，单独运行 `vagrant rsync-auto` )

python+vagrant+virtualbox系列文章<br /> 

- [036-win10搭建python的linux开发环境(pycharm+vagrant+virtualbox)](https://juejin.im/post/5d3a55ece51d454f71439dd2) 
- [037-vagrant启动(up)后自动同步文件(rsync-auto)](https://juejin.im/post/5d562b5e5188252d43756db8) 
- [040-解决Linux使用virtualbox共享文件夹问题](https://juejin.im/post/5d5695056fb9a06afd6600f0)
- [042-解决win10 VirtualBox无法启动(VERR_NEM_VM_CREATE_FAILED)](https://juejin.im/post/5d63869a51882559c41612c6)
- [043-解决vagrant访问virtualbox共享文件夹报无权限问题(Permission denied)](https://juejin.im/post/5d6493d6e51d456206115a2c)

<!-- more -->
<a name="M2lMe"></a>
## sync

```bash
  config.vm.synced_folder ".", "/vagrant", type: "rsync",
    # rsync__verbose: true,
    # rsync__auto: true,
    rsync__exclude: ['.git*', 'node_modules*','*.log','*.box','Vagrantfile']

  config.trigger.after :up do |t|
   t.info = "rsync auto"
   t.run = {inline: "vagrant rsync-auto"}
    # 如果想后台运行，则使用下面语句
    # t.run = {inline: "bash -c 'vagrant rsync-auto &'"}
  end
```

参考 [Vagrant Does not Start RSync-Auto on Up or Reload#briancain's reply](https://github.com/hashicorp/vagrant/issues/10002#issuecomment-419503397)

<a name="3GPil"></a>
## nfs

经测试，在win10上，需要安装插件（vagrant-vbguest vagrant-winnfsd）

```bash
vagrant plugin install vagrant-vbguest vagrant-winnfsd
```
如果报

```bash
Installing the 'vagrant-vbguest' plugin. This can take a few minutes...
ERROR:  SSL verification error at depth 3: unable to get local issuer certificate (20)
ERROR:  You must add /C=US/O=Starfield Technologies, Inc./OU=Starfield Class 2 Certification Authority to your local trusted store
Vagrant failed to load a configured plugin source. This can be caused
by a variety of issues including: transient connectivity issues, proxy
filtering rejecting access to a configured plugin source, or a configured
plugin source not responding correctly. Please review the error message
below to help resolve the issue:

  SSL_connect returned=1 errno=0 state=error: certificate verify failed (https://gems.hashicorp.com/specs.4.8.gz)

Source: https://gems.hashicorp.com/
```
需要设置CAfile

```bash
set SSL_CERT_FILE="path\to\Vagrant\embedded\cacert.pem"
```
如果下载速度慢，并且有境外代理服务器，可以考虑设置代理

```bash
set http_proxy=http://username:password@ip:port
set https_proxy=http://username:password@ip:port
```

设置Vagrantfile
```bash
  # ... 忽略无关内容
  config.vm.synced_folder ".", "/vagrant",
     type:"nfs"
  # ... 忽略无关内容
```

<a name="qruRF"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/08/14/vagrant-startup-run-rsync/)
- [我的掘金](https://juejin.im/post/5d562b5e5188252d43756db8)
- [Vagrant Does not Start RSync-Auto on Up or Reload#briancain's reply](https://github.com/hashicorp/vagrant/issues/10002#issuecomment-419503397)
- [How to speed up Vagrant on Windows 10 using NFS](https://peshmerge.io/how-to-speed-up-vagrant-on-windows-10-using-nfs/)

