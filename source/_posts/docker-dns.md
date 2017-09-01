---
title: 彻底解决docker build时安装软件失败问题
date: 2017-09-01 11:35:31
tags: [docker]
---

最近遇到一个问题，构建Dockerfile镜像时，如果安装软件，有一定概率失败(2%-10%)。以alpine为例

失败日志如下

```bash
Step 4/6 : RUN echo -e "https://mirrors.ustc.edu.cn/alpine/latest-stable/main\nhttps://mirrors.ustc.edu.cn/alpine/latest-stable/community" > /etc/apk/repositories &&     apk update &&     apk add tzdata &&     cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&     echo "Asia/Shanghai" >  /etc/timezone &&     rm -rf /var/cache/apk/*
 ---> Running in bd5d1dfd3ff4
fetch https://mirrors.ustc.edu.cn/alpine/latest-stable/main/x86_64/APKINDEX.tar.gz
fetch https://mirrors.ustc.edu.cn/alpine/latest-stable/community/x86_64/APKINDEX.tar.gz
v3.6.2-83-g1079181bed [https://mirrors.ustc.edu.cn/alpine/latest-stable/main]
v3.6.2-84-g6ee501e465 [https://mirrors.ustc.edu.cn/alpine/latest-stable/community]
OK: 8440 distinct packages available
(1/1) Installing tzdata (2017a-r0)
ERROR: tzdata-2017a-r0: temporary error (try again later)
```


为了重现该问题，简单的构建一个Docker 镜像，基于alpine，安装tzdata，并设置北京时区

为了加速构建，替换为中科大的镜像地址

Dockerfile

```Dockerfile
FROM alpine
RUN echo -e "https://mirrors.ustc.edu.cn/alpine/latest-stable/main\nhttps://mirrors.ustc.edu.cn/alpine/latest-stable/community" > /etc/apk/repositories && \
    apk update &&\
    apk --no-cache add tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" >  /etc/timezone
```

<!-- more -->

其实一开始没有用国内源，用的官方，但是经常失败，以为是墙的问题，辗转换过阿里云镜像，清华镜像，中科大镜像，甚至后来自建镜像 github repo [anjia0532/alpine-package-mirror][] , [三种方法解决docker构建失败(alpine)][],但是都是时好时坏，严重影响效率。

后来在观察nginx访问日志的时候，报错的时候nginx没有产生访问日志，遂怀疑是构建镜像时没有发出网络请求，祭出神器 `tcpdump` 进行进一步排查

**note**

为了减少干扰，实验机器中，没有其他docker服务在跑(否则tcp请求太多)

```bash
sudo tcpdump -i docker0

....

10:47:43.038404 IP6 :: > ff02::16: HBH ICMP6, multicast listener report v2, 1 group record(s), length 28
10:47:43.114734 ARP, Request who-has 172.17.0.1 tell 172.17.0.2, length 28
10:47:43.114746 ARP, Reply 172.17.0.1 is-at 02:42:30:19:53:45 (oui Unknown), length 28
10:47:43.114750 IP 172.17.0.2.48223 > google-public-dns-a.google.com.domain: 18503+ A? alpine.xxx.com. (39)
10:47:43.114775 IP 172.17.0.2.48223 > google-public-dns-b.google.com.domain: 18503+ A? alpine.xxx.com. (39)
10:47:43.114827 IP 172.17.0.2.48223 > google-public-dns-a.google.com.domain: 18687+ AAAA? alpine.xxx.com. (39)
10:47:43.114833 IP 172.17.0.2.48223 > google-public-dns-b.google.com.domain: 18687+ AAAA? alpine.xxx.com. (39)
10:47:43.209679 IP google-public-dns-a.google.com.domain > 172.17.0.2.48223: 18503 1/0/0 A 172.60.20.6 (55)
10:47:43.229261 IP google-public-dns-a.google.com.domain > 172.17.0.2.48223: 18687 0/1/0 (106)

....
```

发现在构建的时候，是走的google的dns进行解析的，因为众多不可描述的问题，google在国内基本是瘫痪状态（google翻译例外）

>Filtering is necessary because all localhost addresses on the host are unreachable from the container’s network. After this filtering, if there are no more nameserver entries left in the container’s /etc/resolv.conf file, the daemon adds public Google DNS nameservers (8.8.8.8 and 8.8.4.4) to the container’s DNS configuration. If IPv6 is enabled on the daemon, the public IPv6 Google DNS nameservers will also be added (2001:4860:4860::8888 and 2001:4860:4860::8844).

原文见官方文档 [Embedded DNS server in user-defined networks][linkEmbeddedDnsServerInUser-defined]

两种方案，

1. 修改宿主机的`hosts`文件，写死ip
2. 修改Docker的`daemon.json`文件

推荐用第二种，参考一下官方文档 [DAEMON CONFIGURATION FILE#On Linux][linkDaemonConfigurationFile#onLinux]

更合理的方案是修改docker的daemon.json `sudo vi /etc/docker/daemon.json`

比如改成dnspod dns

增加 `"dns": ["119.29.29.29"]`

然后`sudo systemctl daemon-reload`

```bash
11:14:17.586559 ARP, Request who-has 172.17.0.1 tell 172.17.0.2, length 28
11:14:17.586577 ARP, Reply 172.17.0.1 is-at 02:42:30:19:53:45 (oui Unknown), length 28
11:14:17.586581 IP 172.17.0.2.43273 > pdns.dnspod.cn.domain: 53616+ A? alpine.xxx.com. (39)
11:14:17.586604 IP 172.17.0.2.43273 > pdns.dnspod.cn.domain: 53868+ AAAA? alpine.xxx.com. (39)
11:14:17.777921 IP pdns.dnspod.cn.domain > 172.17.0.2.43273: 53868 0/1/0 (106)
11:14:17.843875 IP pdns.dnspod.cn.domain > 172.17.0.2.43273: 53616 1/0/0 A 172.60.20.6 (55)
11:14:17.844028 IP 172.17.0.2.36810 > 172.60.20.6.http: Flags [S], seq 4032628285, win 42340, options [mss 1460,sackOK,TS val 1959807306 ecr 0,nop,wscale 11], length 0
```

整个过程可参见我在中科大 github的issues [alpine 镜像频繁异常][linkAlpine镜像频繁异常]

博客 [https://anjia.ml/2017/09/01/docker-dns/][blog]
掘金 [https://juejin.im/post/59a8f9e0f265da24797b7da0][juejin]
简书 [http://www.jianshu.com/p/1f4e62dff251][jianshu]

[blog]: https://anjia.ml/2017/09/01/docker-dns/
[juejin]: https://juejin.im/post/59a8f9e0f265da24797b7da0
[jianshu]: http://www.jianshu.com/p/1f4e62dff251
[anjia0532/alpine-package-mirror]: https://github.com/anjia0532/alpine-package-mirror
[linkDaemonConfigurationFile#onLinux]: https://docs.docker.com/engine/reference/commandline/dockerd/#on-linux
[linkEmbeddedDnsServerInUser-defined]: https://docs.docker.com/engine/userguide/networking/configure-dns/
[linkAlpine镜像频繁异常]: https://github.com/ustclug/discussions/issues/166
[三种方法解决docker构建失败(alpine)]: https://anjia.ml/2017/08/23/alpine-mirror-server/
