---
title: rancher安装Kubernetes
date: 2017-11-10 10:57:14
tags: [k8s,kubernetes,rancher]
---

目前docker官方默认的编排容器改成k8s，已经让k8s成为事实标准，但是受限于天朝的gfw，导致下载`gcr.io` registry的镜像基本没戏。

而rancher中国的两篇博文 [Rancher-k8s加速安装文档][] 和 [原生加速中国区Kubernetes安装][] 我是死活没成功。

本文主要介绍，如何在国内，使用`rancher`加速`k8s`的安装，部分内容也适用于直接原生`k8s`加速



<!--more-->

## 安装 docker

按照 [Getting Started with Hosts#SUPPORTED DOCKER VERSIONS][GettingStartedWithHosts#supported] 安装受支持的`docker-ce version` (如果国内安装较慢，可以考虑使用[中科大docker镜像][] ,或者其他阿里云镜像，腾讯云镜像，清华镜像等)

## 安装rancher
按照 [Installing Rancher Server][InstallingRancherServer] 根据实际情况，安装`rancher` ,建议使用 [加速器 DaoCloud - 业界领先的容器云平台][加速器Daocloud-业界领先的容器云平台] 或者 [阿里云docker加速器][]

## 注册 [docker hub][DockerHub]

## 安装k8s

![](http://ww1.sinaimg.cn/large/afaffa71ly1flcuo8bzfdj210b0ms0u6.jpg)

![](http://ww1.sinaimg.cn/large/afaffa71ly1flcuo8conrj217w0pc76q.jpg)

复制出这串命令，在从机上运行，注册一个主机到k8s环境。稍等大约10分钟左右，基础设施全是绿色。此时`kubernetes-dashboard`是打不开的，提示 `Service unavailable`

按照 群内 `@天阑-李小威` 
![](http://ww1.sinaimg.cn/large/afaffa71ly1flcus1lvxsj20xn0nf0vu.jpg)

给的命令，在`cli`执行 `kubectl --namespace=kube-system get pods`

![](http://ww1.sinaimg.cn/large/afaffa71ly1flcuujqzo1j20oi07874t.jpg)

发现容器一直卡住.

打开`CLI` 运行
```bash
k8s=(
    heapster
    kube-dns
    kubernetes-dashboard
    monitoring-grafana
    monitoring-influxdb
    tiller-deploy
)
for imageName in ${k8s[@]} ; do
    for t in $(kubectl --namespace=kube-system describe deployment $imageName | grep gcr | awk '{print $2}') ; do
        echo $t
    done
done
```
输出类似
```
gcr.io/google_containers/heapster-amd64:v1.3.0-beta.1
gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.5
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.5
gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.5
gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.1
gcr.io/google_containers/heapster-grafana-amd64:v4.0.2
gcr.io/google_containers/heapster-influxdb-amd64:v1.3.3
gcr.io/kubernetes-helm/tiller:v2.3.0
```

找一台能翻墙的vps,`docker login` 登陆docker hub的账号

```bash
#!/usr/bin/env bash

hubName=anjia0532

images=(
    gcr.io/google_containers/heapster-amd64:v1.3.0-beta.1
    gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.5
    gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.5
    gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.5
    gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.1
    gcr.io/google_containers/heapster-grafana-amd64:v4.0.2
    gcr.io/google_containers/heapster-influxdb-amd64:v1.3.3
    gcr.io/kubernetes-helm/tiller:v2.3.0
)

for imageName in ${images[@]} ; do
    imgName=$(echo ${imageName} | cut -d"/" -f3)
    docker pull $imageName
    docker tag $imageName $hubName/$imgName
    docker push $hubName/$imgName
done
```

将`k8s-cli`中输出的版本，替换到`images`中，并修改`hubName`为自己实际的`docker hub` 账号,运行。

输出类似
```
$ docker images
anjia0532/k8s-dns-sidecar-amd64                        1.14.5              fed89e8b4248        6 weeks ago         41.8MB
anjia0532/k8s-dns-kube-dns-amd64                       1.14.5              512cd7425a73        6 weeks ago         49.4MB
anjia0532/k8s-dns-dnsmasq-nanny-amd64                  1.14.5              459944ce8cc4        6 weeks ago         41.4MB
anjia0532/heapster-influxdb-amd64                      v1.3.3              577260d221db        2 months ago        12.5MB
anjia0532/kubernetes-dashboard-amd64                   v1.6.1              71dfe833ce74        5 months ago        134MB
anjia0532/tiller                                       v2.3.0              24d2d8f25332        7 months ago        56MB
anjia0532/heapster-grafana-amd64                       v4.0.2              a1956d2a1a16        9 months ago        131MB
anjia0532/heapster-amd64                               v1.3.0-beta.1       4ff6ad0ca64c        9 months ago        101MB
```


修改`kube-system`的镜像地址,打开cli运行,注意将`anjia0532`替换成`docker hub`账号

```bash
kubectl --namespace=kube-system edit deployment  heapster
kubectl --namespace=kube-system edit deployment  kube-dns
kubectl --namespace=kube-system edit deployment  kubernetes-dashboard
kubectl --namespace=kube-system edit deployment  monitoring-grafana
kubectl --namespace=kube-system edit deployment  monitoring-influxdb
#替换
:%s#gcr.io/google_containers#anjia0532#g
#保存
:wq!

kubectl --namespace=kube-system edit deployment  tiller-deploy
#替换
:%s#gcr.io/kubernetes-helm#anjia0532#g
#保存
:wq!
```

运行
```bash

$ kubectl --namespace=kube-system  get pods
NAME                                    READY     STATUS              RESTARTS   AGE
heapster-2407085140-hgddj               0/1       ContainerCreating   0          48m
kube-dns-570853077-hcqzg                0/3       Pending             0          1h
kube-dns-638003847-8vps9                0/3       ContainerCreating   0          2h
kubernetes-dashboard-3888044391-wm3s3   0/1       ContainerCreating   0          14m
monitoring-grafana-3847008717-06988     0/1       ContainerCreating   0          14m
monitoring-influxdb-3527312529-n3xxw    0/1       ContainerCreating   0          14m
tiller-deploy-402017509-jkw7n           0/1       ContainerCreating   0          13m
```

查看状态，我这边一直`Pending` 手动囧一个,找到原因,后续补充

[GettingStartedWithHosts#supported]: http://rancher.com/docs/rancher/v1.6/en/hosts/#supported-docker-versions
[InstallingRancherServer]: http://rancher.com/docs/rancher/v1.6/en/installing-rancher/installing-server/
[中科大docker镜像]: http://mirrors.ustc.edu.cn/help/docker-ce.html
[加速器Daocloud-业界领先的容器云平台]: https://www.daocloud.io/mirror
[阿里云docker加速器]: https://cr.console.aliyun.com/#/accelerator
[DockerHub]: https://hub.docker.com/
[Rancher-k8s加速安装文档]: https://www.cnrancher.com/rancher-k8s-accelerate-installation-document/
[原生加速中国区Kubernetes安装]: https://www.cnrancher.com/kubernetes-installation/
