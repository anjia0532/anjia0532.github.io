---
title: 057-基于frida的一键脱壳+反编译
urlname: frida-android-decompile
date: '2021-02-04 19:35:20 +0800'
tags:
  - frida
  - android
  - 反编译
  - 脱壳
  - 逆向
categories:
  - python
  - 逆向
---

> 这是坚持技术写作计划（含翻译）的第 57 篇，定个小目标 999，每周最少 2 篇。

开场 冷知识 ，脱壳(qiao)不是脱壳(ke),当然大多数人还是习惯读 ke

基本上能看我这篇的基本都是脱壳一知半解的脚本小子（我自己也是），所以不会讲很高深的（比如 [常见 android app 加固厂商脱壳方法研究](https://cloud.tencent.com/developer/article/1740663)） 。本文主要讲如何安装 frida(国内装 frida 有些坑)和常见脱壳操作。

<!-- more -->

## 安装 Frida

### 安装 Python3 环境

可以自行安装，也可以[参考之前写的安装 anaconda](https://anjia0532.github.io/2017/07/02/anaconda-install-and-configurating-jupyter/)

### 安装 Frida

```python
# pip安装组件
pip install frida frida-tools
```

如果安装 frida 时卡在 `Running setup.py install for frida ... –`  超过 2 分钟，基本会下载失败，此时用别的姿势打开 [https://pypi.org/project/frida/#files](https://pypi.org/project/frida/#files) ，根据实际情况下载对应的 egg,比如你是装的 python3.8，那就搜   `py3.8-win`  比如 [frida-14.2.10-py3.8-win-amd64.egg](https://files.pythonhosted.org/packages/0b/29/1428c995c486a50ec2c83e6ad59e7bff546a90d69618ed1ca7e3bf348ca7/frida-14.2.10-py3.8-win-amd64.egg)
然后 `easy_install frida-14.2.10-py3.8-win-amd64.egg` ,重新执行 `pip install frida frida-tools`

## 安装虚拟机或者准备实体机

本文假设使用虚拟机，比如 [夜神](https://www.yeshen.com/)
打开虚拟机，开启`开发者模式`，启用`usb调试`
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1612424480863-e2240f1c-6652-43cc-aa9a-1c832039bcbf.png#align=left&display=inline&height=899&margin=%5Bobject%20Object%5D&name=image.png&originHeight=899&originWidth=988&size=294765&status=done&style=none&width=988)
打开 cmd，进入夜神安装目录\bin

```bash
# 测试adb是否已连接
xxx\Nox\bin>adb.exe devices
List of devices attached
127.0.0.1:62001 device

# 获取cpu架构
xxx\Nox\bin>adb.exe shell getprop ro.product.cpu.abi
x86

```

## 安装 frida 手机端

[https://github.com/frida/frida/releases](https://github.com/frida/frida/releases) (国内如果慢，可以用[https://hub.fastgit.org/frida/frida/releases](https://hub.fastgit.org/frida/frida/releases) 加速下载)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1612424901964-1983f34f-70fb-4166-9c2d-913335e62aca.png#align=left&display=inline&height=57&margin=%5Bobject%20Object%5D&name=image.png&originHeight=57&originWidth=1046&size=8061&status=done&style=none&width=1046)
解压下载下来的 firda-server-${version}-android-x86.xz，并将 frida-server 移动到虚拟机里，有些 app 会监测 frida 的进程，所以将文件随便命名

```bash
xxx\Nox\bin>adb push frida-server-14.2.10-android-x86 /data/local/tmp/abd
[100%] /data/local/tmp/abd
xxx\Nox\bin>adb shell chmod +x /data/local/tmp/abd && /data/local/tmp/abd
# 转发frida端口
xxx\Nox\bin>adb forward tcp:27042 tcp:27042
xxx\Nox\bin>adb forward tcp:27043 tcp:27043
xxx\Nox\bin>adb forward tcp:38089 tcp:38089
# 启动frida并修改监听端口（防止部分app监测默认端口）
xxx\Nox\bin>adb shell /data/local/tmp/abd -l 0.0.0.0:38089
```

另起一个 cmd，

```bash
# 切换到安装frida 和frida-tools的环境下
activate python3.8
# 测试是否连上frida-server
(python3.8) xxx\Nox\bin>frida-ps -H 127.0.0.1:38089 | findstr net
# 网易云音乐
4336  com.netease.cloudmusic
1892  netd
```

以网易云音乐为例，[https://www.wandoujia.com/apps/293217/history](https://www.wandoujia.com/apps/293217/history)
为啥用豌豆荚下载，而不是别的应用市场，因为豌豆荚支持历史版本，意味着如果新版不好搞，可以多下几个版本，用最低能用版本作为切入点

## 开始脱壳

### 安装

```bash
# 如果端口改了需要clone后手动修改main.py
git clone https://github.com/hluwa/FRIDA-DEXDump
cd FRIDA-DEXDump/frida-dexdump
python main.py -h

# 如果不改端口，可以直接用pip安装
pip install frida-dexdump
frida-dexdump -h
```

如果要修改默认连接地址，修改 [https://github.com/hluwa/FRIDA-DEXDump/blob/master/frida_dexdump/main.py#L177-L183](https://github.com/hluwa/FRIDA-DEXDump/blob/master/frida_dexdump/main.py#L177-L183) 的 connect_device 函数，如果是手机，使用 usb 连接的话，可以不用改

```python
def connect_device(timeout=15):
    manager = frida.get_device_manager()
    device = manager.add_remote_device("127.0.0.1:38089")
    return device
```

### 将夜神 bin 目录配置到环境变量里

如果不配置 adb.exe 到环境变量，会报错

```bash
'adb' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
```

### 脱壳

```bash
(python3.8) xxx\FRIDA-DEXDump\frida_dexdump>python main.py -p 4336

------------------------------------------------------------------------------------------------------------------------
                  ____________ ___________  ___        ______ _______   _______
                  |  ___| ___ \_   _|  _  \/ _ \       |  _  \  ___\ \ / /  _  \
                  | |_  | |_/ / | | | | | / /_\ \______| | | | |__  \ V /| | | |_   _ _ __ ___  _ __
                  |  _| |    /  | | | | | |  _  |______| | | |  __| /   \| | | | | | | '_ ` _ \| '_ \
                  | |   | |\ \ _| |_| |/ /| | | |      | |/ /| |___/ /^\ \ |/ /| |_| | | | | | | |_) |
                  \_|   \_| \_|\___/|___/ \_| |_/      |___/ \____/\/   \/___/  \__,_|_| |_| |_| .__/
                                                                                               | |
                                                                                               |_|
                                      https://github.com/hluwa/FRIDA-DEXDump
------------------------------------------------------------------------------------------------------------------------

02-04/17:25:45 INFO [DEXDump]: found target [4336] com.netease.cloudmusic
'adb' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
[DEXDump]: DexSize=0x8c5e74, DexMd5=ae00d7d709366721d36acd3d9323463c, SavePath=xxx\FRIDA-DEXDump\frida_dexdump/com.netease.cloudmusic/0x9e1aa7dc.dex
```

类似使用 frida 脱壳的方案还有很多，可以参考 [抖音数据采集 Frida 脱壳工具](https://segmentfault.com/a/1190000039075932)

## 反编译

下载 jad-gui [https://github.com/skylot/jadx/releases](https://github.com/skylot/jadx/releases)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1612431111573-32a26128-7ab5-40cd-a089-508ddca70057.png#align=left&display=inline&height=747&margin=%5Bobject%20Object%5D&name=image.png&originHeight=747&originWidth=819&size=110038&status=done&style=none&width=819)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](http://anjia0532.github.io/2021/02/04/frida-android-decompile)
- [我的掘金](https://juejin.cn/post/6925965844191117320/)
- [抖音数据采集 Frida 脱壳工具](https://segmentfault.com/a/1190000039075932)
- [[分享]Android 加固厂商特征](https://bbs.pediy.com/thread-223248.htm)
- [安卓应用的安全和破解](https://crifan.github.io/android_app_security_crack/website/)
