---
title: rancher中国区加速安装Kubernetes
date: 2017-11-13 12:04:14
tags: [k8s,kubernetes,rancher,gcr.io]
---

上篇 [《rancher安装Kubernetes》][] 最后的步骤是错误的，即使每次手动改了k8s的镜像，但是依然服务pull，而且每次重启docker或者k8s，又会重置回默认的`gcr.io`的镜像。

本文是在群内`@天阑-李小威` `@洪晓露` `@logan` 等大神指导下,并根据 [《原生加速中国区Kubernetes安装》][]，最终搞定的方案

<!--more-->


## 环境准备

|    主机名    |     主机ip    |                  OS                  |     docker version    | ranhcer version |
|--------------|---------------|--------------------------------------|-----------------------|-----------------|
| anjia-ubuntu | 192.168.31.83 | ubuntu 17.04 4.9.0-12-generic x86_64 | Docker version 1.12.6 | v1.6.10         |

## 安装 docker

按照 [Getting Started with Hosts#SUPPORTED DOCKER VERSIONS][GettingStartedWithHosts#supported] 安装受支持的`docker-ce version` (如果国内安装较慢，可以考虑使用[中科大docker镜像][] ,或者其他阿里云镜像，腾讯云镜像，清华镜像等)

如果之前装有其他版本的，需要删除所有镜像和容器，并卸载docker重装,rancher k8s 目前只支持 `docker 1.12.3+` 的版本
```bash
sudo apt install docker.io
```

## 安装rancher
按照 [Installing Rancher Server][InstallingRancherServer] 根据实际情况，安装`rancher` ,建议使用 [加速器 DaoCloud - 业界领先的容器云平台][加速器Daocloud-业界领先的容器云平台] 或者 [阿里云docker加速器][]

如果rancher/server是v1.6.10版本(低于v1.6.10版本未试过)，需要你修改私有registry，且将gcr.io的插件push到私有registry，且namespace必须为`google_containers`,如果允许的话，请选择v1.6.11+(目前v1.6.11还是rc版) 请根据实际情况自行选择版本

```bash
sudo docker run -d --restart=unless-stopped --name=rancher-server -p 8080:8080 rancher/server:v1.6.11-rc10 && sudo docker logs -f rancher-server
#sudo docker run -d --restart=unless-stopped --name=rancher-server -p 8080:8080 rancher/server:v1.6.10 && sudo docker logs -f rancher-server
```

## 注册 [docker hub][DockerHub]

## 安装k8s
如果之前安装过docker和k8s，需要运行
```
docker rm -f -v $(docker ps -aq) 
docker volume rm $(docker volume ls)
sudo rm -rf /var/etcd/
```

### 创建环境模板
![创建环境模板](http://ww1.sinaimg.cn/large/afaffa71ly1flglqhdy4yj20860oxmxy.jpg)

### 修改k8s模板

`Private Registry for Add-Ons and Pod Infra Container Image` `index.docker.io`

`Image namespace for  Add-Ons and Pod Infra Container Image` `anjia0532`

`Image namespace for kubernetes-helm Image` `anjia0532`

`Pod Infra Container Image` `anjia0532`

![修改k8s模板](http://ww1.sinaimg.cn/large/afaffa71ly1flglqhebjoj20yt0fpgmd.jpg)

![修改k8s模板](http://ww1.sinaimg.cn/large/afaffa71ly1flglqhfzobj216b0j6gno.jpg)

![修改k8s模板](http://ww1.sinaimg.cn/large/afaffa71ly1flglqhogk9j20we0eujs2.jpg)


### 创建k8s环境

![创建k8s环境](http://ww1.sinaimg.cn/large/afaffa71ly1flglqhhf5sj215y0mljsn.jpg)

### 选择k8s环境并添加主机

![选择k8s环境并添加主机](http://ww1.sinaimg.cn/large/afaffa71ly1flglqhp8dbj217d0q6jtx.jpg)

### 查看k8s基础服务状态

当基础服务都是绿色后，即可使用

![查看k8s基础服务状态](http://ww1.sinaimg.cn/large/afaffa71ly1flgm2741vaj21h70eqdh4.jpg)

### 查看k8s 仪表板 dashboard
![查看k8s 仪表板 dashboard](http://ww1.sinaimg.cn/large/afaffa71ly1flgm274d0bj20s60ahq3d.jpg)

![](http://ww1.sinaimg.cn/large/afaffa71ly1flgm275gqgj21gn0njact.jpg)

## 异常排查

如果打开dashboard 报 `503 ServiceUnavailable` , 非常感谢群内`@天阑-李小威` 耐心解答，同时 参考 [Kubernetes 部署失败的 10 个最普遍原因（Part 1）][Kubernetes部署失败的10个最普遍原因（part1）] 解决了好几个问题

打开`Cli`

```bash
> kubectl --namespace=kube-system  get pods
NAME                                   READY     STATUS             RESTARTS   AGE
heapster-79684d56d6-8pjrd              1/1       Running            0          13m
kube-dns-7f59fd996-nkvv5               3/3       Running            0          13m
kubernetes-dashboard-86d9cc5b4-7lxj5   0/1       ImagePullBackOff   0          13m
monitoring-grafana-6dc7576774-8x79x    1/1       Running            0          13m
monitoring-influxdb-d78f84c6c-29wcp    1/1       Running            0          13m
tiller-deploy-c4598db7d-8wxpp          1/1       Running            0          13m

# 复
> kubectl --namespace=kube-system  describe pod kubernetes-dashboard-86d9cc5b4-7lxj5
# 我这是正常Running的日志,ImagePullBackOff的没截下来
 Events:
  Type    Reason                 Age   From               Message
  ----    ------                 ----  ----               -------
  Normal  Scheduled              16m   default-scheduler  Successfully assigned kubernetes-dashboard-86d9cc5b4-7lxj5 to k8s
  Normal  SuccessfulMountVolume  16m   kubelet, k8s       MountVolume.SetUp succeeded for volume "io-rancher-system-token-lb68r"
  Normal  Pulled                 16m   kubelet, k8s       Container image "index.docker.io/anjia0532/kubernetes-dashboard-amd64:v1.7.1" already present on machine
  Normal  Created                16m   kubelet, k8s       Created container
  Normal  Started                16m   kubelet, k8s       Started container

# 也可以根据 events 来辅助排查问题
> kubectl --namespace=kube-system get events
```


博客 [https://anjia.ml/2017/11/13/rancher-k8s-china/][blog]
掘金 [https://juejin.im/post/5a097599f265da430d578385][juejin]
简书 [http://www.jianshu.com/p/2f906a7f4bfa][jianshu]

[blog]: https://anjia.ml/2017/11/13/rancher-k8s-china/
[juejin]: https://juejin.im/post/5a097599f265da430d578385
[jianshu]: http://www.jianshu.com/p/2f906a7f4bfa
[GettingStartedWithHosts#supported]: http://rancher.com/docs/rancher/v1.6/en/hosts/#supported-docker-versions
[InstallingRancherServer]: http://rancher.com/docs/rancher/v1.6/en/installing-rancher/installing-server/
[中科大docker镜像]: http://mirrors.ustc.edu.cn/help/docker-ce.html
[加速器Daocloud-业界领先的容器云平台]: https://www.daocloud.io/mirror
[阿里云docker加速器]: https://cr.console.aliyun.com/#/accelerator
[DockerHub]: https://hub.docker.com/
[Rancher-k8s加速安装文档]: https://www.cnrancher.com/rancher-k8s-accelerate-installation-document/
[原生加速中国区Kubernetes安装]: https://www.cnrancher.com/kubernetes-installation/
[《rancher安装Kubernetes》]: https://anjia.ml/2017/11/10/rancher-k8s/
[《原生加速中国区Kubernetes安装》]: https://www.cnrancher.com/kubernetes-installation/
[Kubernetes部署失败的10个最普遍原因（part1）]: http://dockone.io/article/2247
