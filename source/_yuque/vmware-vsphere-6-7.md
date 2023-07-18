---
title: '024-VMWare VSphere 6.7(ESXI,VSCA) 下载'
urlname: vmware-vsphere-6-7
date: '2019-05-20 21:30:00 +0800'
tags:
  - 云计算
  - 虚拟化
  - 企业级虚拟化
  - 私有云
categories: 云计算
---

> 这是坚持技术写作计划（含翻译）的第 24 篇，定个小目标 999，每周最少 2 篇。

VSphere 是一套组件的合成，类似，word,excel,ppt 合称 office。而 ESXI 是虚拟组件，虚拟机跑在 ESXI 上，而 VSCA 是 vcenter,是一个集群管理软件。提供企业级功能，比如虚拟机高可用等。

本文主要讲解如何下载 ESXI 和 VSCA。

## 下载 ESXI

1. 转至   [VMware vSphere Hypervisor（ESXi）6.7 下载页面](https://my.vmware.com/web/vmware/evalcenter?p=free-esxi6)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558333257691-7387ac13-94fc-43fb-b29d-9ff232ca77c9.png#align=left&display=inline&height=531&originHeight=531&originWidth=1151&size=70459&status=done&width=1151)

2. 登陆或者创建一个 VMware 账号
3. ![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558333321551-eae0c876-c98e-484c-8e4d-bb9b623bf3e1.png#align=left&display=inline&height=516&originHeight=516&originWidth=1033&size=57459&status=done&width=1033)
4. 下载最新的 6.7.0U2。
5. 安装到硬件设备上
6. 如果只是个人用，则可以使用上图打马赛克的个人许可证。

## VSCA

vcenter 是企业套件，无法通过官网免费下载。通过 google 搜索得到
[https://technet24.ir/vmware-vcenter-server-6-7-13940](https://technet24.ir/vmware-vcenter-server-6-7-13940) 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558334332060-23ee4240-2db2-4a0f-bcf8-0bc3bb3a7bcb.png#align=left&display=inline&height=771&originHeight=771&originWidth=755&size=147702&status=done&width=755)
其实  technet24.ir 也提供 EXSI ISO 的下载方式

注意，为了安全起见，下载后，需要去跟官网的摘要进行比对，
[https://my.vmware.com/cn/group/vmware/details?downloadGroup=VC67U2A&productId=742#errorCheckDiv](https://my.vmware.com/cn/group/vmware/details?downloadGroup=VC67U2A&productId=742#errorCheckDiv)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558334892852-4ff231a2-498f-4795-aadc-9cc7e20ae821.png#align=left&display=inline&height=294&originHeight=294&originWidth=1480&size=49917&status=done&width=1480)

安装和使用方式，参考  [Install VCSA 6.7.1](https://blog.51cto.com/happynews/2312006)

## 参考资料

- [VMware vSphere Hypervisor（ESXi）6.7 下载页面](https://my.vmware.com/web/vmware/evalcenter?p=free-esxi6)
- [Vmware ESXi upgrade 11 月 9 日](https://blog.51cto.com/happynews/2316683?source=dra)
- [Install VCSA 6.7.1](https://blog.51cto.com/happynews/2312006)
- [VMware VCenter Server 6.7 U2a](https://technet24.ir/vmware-vcenter-server-6-7-13940)
- [VMware ESXi Patch Tracker](https://esxi-patches.v-front.de/ESXi-6.7.0.html)
- [VMware ESXi Image Profiles](https://www.virten.net/vmware/vmware-esxi-image-profiles/)
- [vmware 6 虚拟化 全系列 序列号](http://www.i5i6.net/post/190.html)
