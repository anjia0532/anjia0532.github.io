---
title: synergy 一套键鼠多台设备共享
date: 2017-02-08 17:44:28
tags: [synergy]
categories: [工具]
---
Synergy 可以在多台电脑之间共享鼠标、键盘、剪贴板。开源，跨 Win、Linux、Mac。

<!-- more -->

Synergy 需要注意不是远控软件，类似双屏或者KVM切换器，只是共享鼠标和键盘

具体关于synergy的介绍可以看 [Synergy 一套键鼠同时控制多台电脑的神器！超级方便！开源免费，支持\(Win/Mac/Linux\)](http://www.iplaysoft.com/synergy.html) [Synergy – 教你在局域网中用一套键盘/鼠标控制多台电脑](http://www.appinn.com/synergy/)

>想必很多人都拥有多台电脑，譬如台式机+笔记本，很多时候我们都会同时打开它们工作。可是你有没发现，如果桌子上摆放着多台电脑多套键盘鼠标，不停来回切换使用是否很累呢？如果说现在可以只用一套键鼠，就能同时控制你全部的电脑，你会否兴奋？

>Synergy 正是为此而生的好工具！它可以让你的多台电脑共享一套键鼠，甚至还可以共享剪贴板，而你只需动动鼠标，指针就可以轻松地在各台电脑屏幕之间来回穿梭，就像一台电脑使用多个显示器一样。而且 Synergy 完全免费开源，并跨平台支持 Win/Mac/Linux，相当给力！ 使用之后，工作效率提高，腿不酸腰不疼，桌面也干净了，绝对是绝世神器啊！

但是该文章中的版本较旧，本着折腾的态度，终于搞定2台PC（Ubuntu Zesty Zapus + Windows10 1607版），安装最新版并且免费使用。


## 注册[synergy](https://symless.com/synergy/#get-synergy)

填写`email`和`fullname`

<kbd>Enter</kbd>回车确定，不进行支付

## 下载最新版本[每夜构建版本](https://symless.com/nightly)

按照需要下载指定版本

比如我下载的 [synergy-master-stable-a5140aa-Windows-x64.msi](https://symless.com/files/nightly/synergy-master-stable-a5140aa-Windows-x64.msi) 和 [synergy-master-stable-a5140aa-Linux-x86_64.deb](https://symless.com/files/nightly/synergy-master-stable-a5140aa-Linux-x86_64.deb)

## 获取序列号

从 [1.4.18免费下载页面](http://symless.com/download/free/)随便下载一个，即可显示序列号，复制并保存下来，如果没有，则点击右上角的`Login` 进行登录，重复此步骤

## 安装windows版本

**千万注意安装和运行时，退出360，否则会卡死，妈的，被坑的很惨**

![360sb](https://ooo.0o0.ooo/2017/02/09/589bcf5446e43.png)

### 安装步骤

![synergy.png](https://ooo.0o0.ooo/2017/02/08/589af1a1280b0.png)

有个地方选择语言，因为已经安装过了，无法截图，可参见 http://www.veryhuo.com/down/html/90189.html

### 设置服务器

![synergy2.png](https://ooo.0o0.ooo/2017/02/08/589af40123cb3.png)

## 安装Ubuntu

```bash

# synergy 依赖 libavahi-compat-libdnssd1
# 但是从sudo apt install -y libavahi-compat-libdnssd1 会提示找不到已废弃
# 所以手动下载
# 从https://www.ubuntuupdates.org/package_metas?utf8=%E2%9C%93&q=libavahi-compat-libdnssd1 找最新的下载

wget http://security.ubuntu.com/ubuntu/pool/main/a/avahi/libavahi-compat-libdnssd1_0.6.32-1ubuntu1_amd64.deb #ubuntu 16.10 版本

wget https://symless.com/files/nightly/synergy-master-stable-a5140aa-Linux-x86_64.deb

sudo dpkg -i libavahi-compat-libdnssd1_0.6.32-1ubuntu1_amd64.deb

# 此时如果提示依赖项未安装，则执行
# sudo apt-get update # 更新
# sudo apt-get -f install # 解决依赖关系
# sudo dpkg -i xxx.deb # 重新安装

sudo dpkg -i synergy-master-stable-a5140aa-Linux-x86_64.deb

nohup synergy &
```

![2017-02-09 09-40-58屏幕截图.png](https://ooo.0o0.ooo/2017/02/09/589bcf5467e12.png)

synergy启动后取消自动配置，手动填写server ip

**注意如果在server端未设置client，client会一直报错**


**client和server需要在一个局域网里，否则无法连接。如果网速慢的话，server控制client会出现卡顿现象**

**如果在一个局域网但是不是一个网段，无法直接ping通可以通过端口映射e.g.  Ngrok等软件进行端口映射 **