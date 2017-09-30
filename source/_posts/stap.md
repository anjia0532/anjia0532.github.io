---
title: openresty使用火焰图排查性能问题
date: 2017-09-12 16:31:40
tags: [openresty,stap,systemtap,flame-graph]
---

本文主要是讲解如何在ubuntu安装最新Systemtap.以及绘制火焰图

## 安装调试镜像

```bash

# 导入 GPG key
# 16.04 and higher
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C8CAB6595FDFF622

#older distributions
#sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ECDCAD72428D7C01 

# 设置源
codename=$(lsb_release -c | awk  '{print $2}')
sudo tee /etc/apt/sources.list.d/ddebs.list << EOF
deb http://ddebs.ubuntu.com/ ${codename}      main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-security main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-updates  main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-proposed main restricted universe multiverse
EOF

# 更新
sudo apt-get update

# 安装调试镜像
sudo apt-get install -y linux-image-$(uname -r)-dbgsym

```

<!--more-->

## 安装最新版 systemtap

```
$ sudo apt-get install -y build-essential zlib1g-dev elfutils libdw-dev gettext

# https://sourceware.org/elfutils/ftp/?C=M;O=D
$ wget https://sourceware.org/elfutils/ftp/0.170/elfutils-0.170.tar.bz2
$ tar xf elfutils-0.170.tar.bz2

# https://sourceware.org/systemtap/ftp/releases/?C=M;O=D
$ wget https://sourceware.org/systemtap/ftp/releases/systemtap-3.1.tar.gz
$ tar zxf systemtap-3.1.tar.gz

$ cd systemtap-3.1

$ ./configure --prefix=/opt/stap --disable-docs \
    --disable-publican --disable-refdocs CFLAGS="-g -O2" \
    --with-elfutils=../elfutils-0.170

$ make -j$(getconf _NPROCESSORS_ONLN) && sudo make install

# export STAP_HOME=/opt/stap/
# export PATH=$STAP_HOME:$PATH

# stap -V

Systemtap translator/driver (version 3.1/0.170, non-git sources)
Copyright (C) 2005-2017 Red Hat, Inc. and others
This is free software; see the source for copying conditions.
tested kernel versions: 2.6.18 ... 4.10-rc8
enabled features: PYTHON2 PYTHON3 LIBXML2 NLS READLINE

```

## 测试是否生效

```
# stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'
Pass 1: parsed user script and 465 library scripts using 77388virt/46648res/5256shr/41840data kb, in 80usr/30sys/333real ms.
Pass 2: analyzed script: 1 probe, 1 function, 7 embeds, 0 globals using 260440virt/231204res/6736shr/224892data kb, in 1680usr/350sys/7050real ms.
Pass 3: translated to C into "/tmp/stap8Lyxq5/stap_e1c4934460a3e749f6deefe95dd50015_2729_src.c" using 260440virt/231404res/6936shr/224892data kb, in 10usr/0sys/5real ms.
Pass 4: compiled C into "stap_e1c4934460a3e749f6deefe95dd50015_2729.ko" in 5260usr/420sys/7185real ms.
Pass 5: starting run.
read performed
Pass 5: run completed in 0usr/20sys/486real ms.

```

## 绘制火焰图

### 下载各工具包
```
# git clone https://github.com/openresty/stapxx.git --depth=1 /opt/stapxx
# export STAP_PLUS_HOME=/opt/stapxx
# export PATH=$STAP_PLUS_HOME:$STAP_PLUS_HOME/samples:$PATH

# stap++ -e 'probe begin { println("hello") exit() }'

hello


# git clone https://github.com/openresty/openresty-systemtap-toolkit.git --depth=1 /opt/openresty-systemtap-toolkit

# git clone https://github.com/brendangregg/FlameGraph.git --depth=1 /opt/FlameGraph
```

### 绘制火焰图
```
# /opt/stapxx/samples/lj-lua-stacks.sxx --arg time=120 --skip-badvars -x `ps --no-headers -fC nginx|awk '/worker/  {print$2}'| shuf | head -n 1` > /tmp/tmp.bt （-x 是要抓的进程的 pid， 探测结果输出到 tmp.bt）
# /opt/openresty-systemtap-toolkit/fix-lua-bt tmp.bt > /tmp/flame.bt  (处理 lj-lua-stacks.sxx 的输出，使其可读性更佳)
# /opt/FlameGraph/stackcollapse-stap.pl /tmp/flame.bt > /tmp/flame.cbt
# /opt/FlameGraph/flamegraph.pl /tmp/flame.cbt > /tmp/flame.svg
```

为了突出效果，建议在运行`stap++`的时候，使用压测工具，以便采集足够的样本

```
# ab -n 10000 -c 100 -k http://localhost/
```

用浏览器打开 `/tmp/flame.svg` 尽量用 `chrome` `firefox`别用国产乱七八糟浏览器.

## openresty/stapxx

```
## 使用 stap++ --args xx.sxx查看具体参数

# stap++ --args /opt/stapxx/samples/lj-lua-stacks.sxx
    --arg depth=VALUE (default: 30)
    --arg detailed=VALUE (default: 0)
    --arg limit=VALUE (default: 1000)
    --arg min=VALUE (default: 2)
    --arg nointerp=VALUE (default: )
    --arg nojit=VALUE (default: )
    --arg probe=VALUE (default: timer.profile)
    --arg time=VALUE (default: )
```

具体脚本用法，参见 [openresty/stapxx#samples][]

## openresty/openresty-systemtap-toolkit

这一系列脚本很有用，比如可以用来看共享内存大小，使用情况，内存泄露情况，哪里泄露的，不过部分脚本需要在编译的时候，开启调试或者增加依赖。具体参见[readme][].

如果要使用`ngx-leaked-pools`需要用到`dtrace`
```bash
$ apt install systemtap-sdt-dev -y
$ ./configure --prefix=/etc/openresty \
  --with-dtrace-probes
```

如果要用到`ngx-pcrejit`需要在编译openresty时增加`--with-pcre-opt=-g`

重新编译并`make && make install` 后会将原有的二进制文件重命名为`${openresty_home}/nginx/sbin/nginx.old`，并创建一个`${openresty_home}/nginx/sbin/nginx`(新版)

```bash
$ kill -USR2 `cat /var/run/nginx.pid`
```

通过`ps -fC nginx`或者`ps -fC openresty`查看新版本是否成功启动

如果成功启动，此时新旧版本同时接受请求

通过
```
$ kill -QUIT `cat /var/run/nginx.pid.oldbin`
```

平滑杀掉旧版


更多资料请自行谷歌、百度。或者参阅 下面的**参考连接**

## 参考连接 

- [白话火焰图-火丁笔记][]
- [Build Systemtap-openresty官方文档][linkBuildSystemtap-openresty官方文档]
- [火焰图-openresty最佳实践][]
- [Systemtap - ubuntu wiki][Systemtap-UbuntuWiki]
- [openresty/stapxx][]
- [openresty/openresty-systemtap-toolkit][]
- [brendangregg/FlameGraph][]
- [虢兆坤- Nginx 的启动、停止、平滑重启、信号控制和平滑升级][虢兆坤-Nginx的启动、停止、平滑重启、信号控制和平滑升级]


博客 [https://anjia.ml/2017/09/12/stap/][blog]
掘金 [https://juejin.im/post/59ce27fef265da065b66d54b][juejin]
简书 [http://www.jianshu.com/p/008fde8837f5][jianshu]

[blog]: https://anjia.ml/2017/09/12/stap/
[juejin]: https://juejin.im/post/59ce27fef265da065b66d54b
[jianshu]: http://www.jianshu.com/p/008fde8837f5


[白话火焰图-火丁笔记]: https://huoding.com/2016/08/18/531
[linkBuildSystemtap-openresty官方文档]: http://openresty.org/en/build-systemtap.html
[火焰图-openresty最佳实践]: https://moonbingbing.gitbooks.io/openresty-best-practices/content/flame_graph.html
[Systemtap-UbuntuWiki]: https://wiki.ubuntu.com/Kernel/Systemtap
[openresty/stapxx]: https://github.com/openresty/stapxx/blob/master/README.markdown
[openresty/openresty-systemtap-toolkit]: https://github.com/openresty/openresty-systemtap-toolkit/blob/master/README.markdown
[brendangregg/FlameGraph]: https://github.com/brendangregg/FlameGraph/blob/master/README.md
[openresty/stapxx#samples]: https://github.com/openresty/stapxx#samples
[readme]: https://github.com/openresty/openresty-systemtap-toolkit/#tools
[虢兆坤-Nginx的启动、停止、平滑重启、信号控制和平滑升级]: http://zachary-guo.iteye.com/blog/1358312
