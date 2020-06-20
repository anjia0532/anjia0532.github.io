---
title: 045-解决packetbeat静默安装pcap问题
urlname: win-packetbeat-silent-pcap
date: 2019-09-22 20:13:21 +0800
tags: [windows,beats,elastic,监控,wireshark,winpcap,pcap,silent-install]
categories: [监控,运维]
---

> 这是坚持技术写作计划（含翻译）的第 45 篇，定个小目标 999，每周最少 2 篇。

在 windows 下安装[elastic packetbeat](https://www.elastic.co/guide/en/beats/packetbeat/current/packetbeat-installation.html#win)  时   要求安装 [pcap](https://github.com/the-tcpdump-group/libpcap) ，而团队内使用 ansible 同一部署，所以需要研究一下静默安装 pcap

<!-- more -->

ansible-playbook

```yaml
- name: 安装Nmap依赖
  win_package:
    path: "https://nmap.org/dist/nmap-7.12-setup.exe"
    product_id: nmap
    arguments:
      - /S
      - /winpcap_mode=yes
```

或者使用命令行安装
下载  [nmap-7.12-setup](https://nmap.org/dist/nmap-7.12-setup.exe)

```bash
/path/to/nmap-7.12-setup.exe /S /winpcap_mode=yes
```

捎带手打个广告  [https://github.com/anjia0532/ansible-beats](https://github.com/anjia0532/ansible-beats) fork 自 elastic 官方的 beats playbook，并增加了 windows 支持

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](http://anjia0532.github.io/2019/09/22/win-packetbeat-silent-pcap)
- [我的掘金](http://juejin.im/post/5d8897b7e51d45620541048d)
- [Winpcap - How to get silent installation back?](https://www.reddit.com/r/sysadmin/comments/71udhh/winpcap_how_to_get_silent_installation_back/)
