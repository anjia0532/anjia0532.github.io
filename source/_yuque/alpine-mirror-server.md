---
title: alpine-mirror-server
date: 2017-08-23 08:49:53
tags: [docker,alpine,alpine-mirror]
---

alpine linux是一个最小化linux系统，常用作docker基础镜像。可以有效减小镜像体积

但是天朝网络很。。。。所以经常容易安装软件失败(`apk update && apk --no-cache add ...`)

<!--more-->

## 利用国内镜像源

[清华镜像][]

[中科大镜像][]

[阿里云镜像][]

三个都用过，但是都会出现安装软件失败的情况(需要多次重新构建)，严重影响效率。

## 境外服务器做反代

如果有幸有台境外(东京，香港等)服务器，又不想镜像站(全部镜像下载)，可以考虑使用nginx反代国外镜像(找一个近源高质量镜像，别三天两头老崩溃的那种)

## 自建镜像站

截止 20170510  官方给出的全部镜像的磁盘使用量

|  edge |  v2.4 |  v2.5 |  v2.6 |  v2.7 |  v3.0 |  v3.1 |  v3.2 |  v3.3 |  v3.4 |  v3.5 |  v3.6 | Total  |
|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|--------|
| 53.1G | 18.8G | 10.4G | 13.0G | 16.5G | 16.5G | 17.5G | 14.5G | 19.0G | 23.2G | 32.5G | 34.4G | 269.5G |

一般自用的话，只会用有限几个版本，比如`v3.6` 的`x86_64` ?那么其余的完全可以忽略，这么一来会小很多，大约11G左右。

核心命令是

rsync.sh

```bash
src=rsync://rsync.alpinelinux.org/alpine/ 
dest=/usr/share/nginx/html

/usr/bin/rsync -prua \
    --exclude-from /etc/rsync/exclude.txt \
    --delete \
    --timeout=600 \
    --delay-updates \
    --delete-after \
    "$src" "$dest"
```
/etc/rsync/exclude.txt
```
edge/
v2.*/
v3.0/
v3.1/
v3.2/
v3.3/
v3.4/
v3.5/
aarch64/
armhf/
ppc64le/
s390x/
x86/
```

解释一下,edge+v*.* 是版本号，其余的是不同cpu架构的不同版本。x86是intel 的32位

```bash
lscpu | grep Architecture
```

根据实际情况，自行加减

详细情况，详见项目 [anjia0532/alpine-package-mirror](https://github.com/anjia0532/alpine-package-mirror)

博客 [https://anjia.ml/2017/08/23/alpine-mirror-server/][blog]
掘金 [https://juejin.im/post/599b1b2a51882511264e7097][juejin]
简书 [http://www.jianshu.com/p/36396a20ea4c][jianshu]

[blog]: https://anjia.ml/2017/08/23/alpine-mirror-server/
[juejin]: https://juejin.im/post/599b1b2a51882511264e7097
[jianshu]: http://www.jianshu.com/p/36396a20ea4c
[阿里云镜像]: https://mirrors.aliyun.com/alpine/
[中科大镜像]: https://mirrors.ustc.edu.cn/alpine/
[清华镜像]: https://mirrors.tuna.tsinghua.edu.cn/alpine/
