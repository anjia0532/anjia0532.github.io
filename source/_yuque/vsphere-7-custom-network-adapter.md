---
title: 050-vSphere7添加第三方网卡驱动（以realtek 8168为例）
urlname: vsphere-7-custom-network-adapter
date: 2020-06-19 23:21:35 +0800
tags: [vsphere,realtek,螃蟹卡,exsi]
categories: [虚拟化]
---

> 这是坚持技术写作计划（含翻译）的第 50 篇，定个小目标 999，每周最少 2 篇。

趁着 618 攒了台 NAS,配置 AMD 2200G(带核显 Vega8)+华擎 A320M-HDV R4.0+8G 酷兽 DDR4 2666MHz\*2+七彩虹 CN600 128G nvme +家里闲置垃圾盘 这配置高不成低不就的，既没有成品 NAS 的低功耗，又没有垃圾佬心头肉(蜗牛星际，玩客云，暴风酷云二期等）便宜，做家用机又略显不足。

本文主要记录如何华擎自带螃蟹卡 RealTek 8168 网卡驱动添加到 vSphere7 里，如果你觉得你头铁直接安装的话，100%会是
![](https://cdn.nlark.com/yuque/0/2020/png/226273/1592623396796-df8ad897-f6ea-48e7-bc15-bbf1c8053ff9.png#align=left&display=inline&height=292&margin=%5Bobject%20Object%5D&originHeight=292&originWidth=661&size=0&status=done&style=none&width=661)

<!-- more -->

官方建议的是 vSphere ESXI Image Builder ,但是太底层了，比较难搞，好在有别人搞的第三方工具， ESXI-Customizer.

## 下载适合 VMWare ESXI 的网卡驱动程序

此处以 Realtek 8168 为例，可以尝试从 google 或者 bing 国际搜 `Realtek NIC drivers for ESXI`  或者直接去 v-front 站找一下  [List of currently available ESXi packages](https://vibsdepot.v-front.de/wiki/index.php/List_of_currently_available_ESXi_packages) 直接 CTRL+F 搜具体型号，比如 [8168](https://vibsdepot.v-front.de/wiki/index.php/Net55-r8168#Direct_Download_links)
如果找到了，直接点击超链接，最后面，从 `Direct Download links` 下载驱动

`.vib.gz`  或者 `.zip`  二选一就行
![image.png](https://cdn.nlark.com/yuque/0/2020/png/226273/1592623916316-f0041989-e8bf-4820-808a-9365abb563d6.png#align=left&display=inline&height=605&margin=%5Bobject%20Object%5D&name=image.png&originHeight=605&originWidth=690&size=76654&status=done&style=none&width=690)

## 使用 ESXi-Customizer-PS 添加驱动

同样还是 v-front 站 [ESXi-Customizer-PS](https://www.v-front.de/p/esxi-customizer-ps.html) 但是官方的注意事项说了，该页面已过期，不再维护，转战[gayhub](https://raw.githubusercontent.com/VFrontDe/ESXi-Customizer-PS/master/ESXi-Customizer-PS.ps1)了，而且支持最新的 7.0(网上常见的 ESXi-Customizer-PS 2.6.0 只到 6.7)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/226273/1592624163515-cc440bcb-5288-44d5-ad9f-08bd8f12e531.png#align=left&display=inline&height=390&margin=%5Bobject%20Object%5D&name=image.png&originHeight=390&originWidth=719&size=67632&status=done&style=none&width=719)
但是用这个脚本有两点限制

1. VMWare PowerCLI5.1+ `Install-Module -Name VMware.PowerCLI [-AllowClobber] [-Proxy http://ip:port]` （毕竟从国外下载，PowerCLI 一共 320Mb+,如果有代理的话，可选的用 -Proxy 来加速下载， `-AllowClobber`  忽略警告，针对于，网络不稳定断开后，不加会提示本地已存在）
1. 修改 PS 的安全策略 `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned`

![image.png](https://cdn.nlark.com/yuque/0/2020/png/226273/1592624767275-3d1c6fe3-dc71-44ff-bc0e-086fe620b1cf.png#align=left&display=inline&height=516&margin=%5Bobject%20Object%5D&name=image.png&originHeight=516&originWidth=839&size=56037&status=done&style=none&width=839)

众所周知的原因，访问国外的网络一向不太稳定， 所以，本次构建直接没有基于 vft,而是自己下驱动和 VMware vSphere Hypervisor (ESXi 7) Offline Bundle 包
从官网或者 [https://technet24.ir/vmware-vsphere-7-21701](https://technet24.ir/vmware-vsphere-7-21701) 下载
![image.png](https://cdn.nlark.com/yuque/0/2020/png/226273/1592628422300-4509bc80-0f0a-40b8-99af-faf0eb738c26.png#align=left&display=inline&height=767&margin=%5Bobject%20Object%5D&name=image.png&originHeight=767&originWidth=981&size=143138&status=done&style=none&width=981)
非官网下载的文件，还是小心验证 MD5 和 SHA1 为妙。这是个好习惯。
![image.png](https://cdn.nlark.com/yuque/0/2020/png/226273/1592628530602-5246c62b-449e-486e-8e99-8d368ab5592f.png#align=left&display=inline&height=565&margin=%5Bobject%20Object%5D&name=image.png&originHeight=565&originWidth=1062&size=110422&status=done&style=none&width=1062)
`.\ESXi-Customizer-PS.ps1 -izip .\VMware-ESXi-7.0.0-15843807-depot.zip -dpt .\net55-r8168-8.045a-napi-offline_bundle.zip -load net55-r8168`

如果是多个网卡驱动，把驱动放到 `D:\xxx\xx`  文件夹下
`.\ESXi-Customizer-PS.ps1 -izip .\VMware-ESXi-7.0.0-15843807-depot.zip -pkgDir D:\xxx\xx`

如果是全局挂代理了，或者属于肉身 FQ 的，可以直接在线
`.\ESXi-Customizer-PS.ps1 -v70 -vft -load net55-r8168`

```powershell
PS D:\xxx\Tools\VMware\PowerCLI> .\ESXi-Customizer-PS.ps1 -izip .\VMware-ESXi-7.0.0-15843807-depot.zip -dpt .\net55-r8168-8.045a-napi-offline_bundle.zip -load net55-r8168
This is ESXi-Customizer-PS Version 2.8.0 (visit https://ESXi-Customizer-PS.v-front.de for more information!)
(Call with -help for instructions)

Logging to C:\Users\xxx\AppData\Local\Temp\ESXi-Customizer-PS-28980.log ...

Running with PowerShell version 5.1 and VMware PowerCLI version .. build

Adding base Offline bundle .\VMware-ESXi-7.0.0-15843807-depot.zip ... [OK]

Connecting additional depot .\net55-r8168-8.045a-napi-offline_bundle.zip ... [OK]

Getting Imageprofiles, please wait ... [OK]

Using Imageprofile ESXi-7.0.0-15843807-standard ...
(Dated 03/16/2020 10:48:54, AcceptanceLevel: PartnerSupported,
The general availability release of VMware ESXi Server 7.0.0 brings whole new levels of virtualization performance to datacenters and enterprises.)

Load additional VIBs from Online depots ...
   Add VIB net55-r8168 8.045a-napi [New AcceptanceLevel: CommunitySupported] [OK, added]

Exporting the Imageprofile to 'D:\xxx\Tools\VMware\PowerCLI\ESXi-7.0.0-15843807-standard-customized.iso'. Please be patient ...

警告: The image profile fails validation.  The ISO / Offline Bundle will still be generated but may contain errors and
may not boot or be functional.  Errors:
警告:   VIB Realtek_bootbank_net55-r8168_8.045a-napi requires vmkapi_2_2_0_0, but the requirement cannot be satisfied
within the ImageProfile.
警告:   VIB Realtek_bootbank_net55-r8168_8.045a-napi requires com.vmware.driverAPI-9.2.2.0, but the requirement cannot
be satisfied within the ImageProfile.

All done.
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/226273/1592627879405-99d6b8fa-b918-44f9-a705-9f932d3a8245.png#align=left&display=inline&height=506&margin=%5Bobject%20Object%5D&name=image.png&originHeight=506&originWidth=839&size=55071&status=done&style=none&width=839)

## 做启动 U 盘

使用 Rufus (用软碟通 UltraISO） 经常会不识别，
![image.png](https://cdn.nlark.com/yuque/0/2020/png/226273/1592630388966-aa5c6704-3582-4f7c-bab3-6f6db9f26ca0.png#align=left&display=inline&height=580&margin=%5Bobject%20Object%5D&name=image.png&originHeight=580&originWidth=418&size=32756&status=done&style=none&width=418)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 加入我们 , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2020/06/19/vsphere-7-custom-network-adapter/)
- 我的掘金
- [024-VMWare VSphere 6.7(ESXI,VSCA) 下载](https://juejin.im/post/5d0b5f8ef265da1b855c5cb8)
- [VMware ESXi 7.0.0 服务器虚拟化](https://www.yuangezhizao.cn/articles/VM/ESXi/init.html)
- [PowerCLI Offline Installation Walkthrough](https://blogs.vmware.com/PowerCLI/2018/01/powercli-offline-installation-walkthrough.html)
- [【更新】【下载】VMware vSphere 7.0 正式版](https://www.azurew.com/vmware/4849.html)
