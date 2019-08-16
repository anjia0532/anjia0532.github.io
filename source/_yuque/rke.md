---
title: rke安装k8s
date: 2018-01-05 16:56:28
tags: [rancher,rke,k8s,kubernetes]
---

> 安装Kubernetes是公认的对运维和DevOps而言最棘手的问题之一。因为Kubernetes可以在各种平台和操作系统上运行，所以在安装过程中需要考虑很多因素。
>
> 在这篇文章中，我将介绍一种新的、用于在裸机、虚拟机、公私有云上安装Kubernetes的轻量级工具——Rancher Kubernetes Engine（RKE）。RKE是一个用Golang编写的Kubernetes安装程序，极为简单易用，用户不再需要做大量的准备工作，即可拥有闪电般快速的Kubernetes安装部署体验。

<!--more-->

Rancher Kubernetes Engine (RKE) 旨在简化k8s的安装。但是根据官方blog和github文档，安装遇到很多问题，遂进行记录，以备查阅。

环境准备(node必须2个+，之前用一个node会报 certificate signed by unknown authority)

| 主机名        | 主机ip         | OS                                   | docker version |
| ---------- | ------------ | ------------------------------------ | -------------- |
| k8s-server | 172.60.20.12 | ubuntu 17.04 4.9.0-12-generic x86_64 | 17.03.2-ce     |
| k8s-worker | 172.60.20.13 | ubuntu 17.04 4.9.0-12-generic x86_64 | 17.03.2-ce     |

## 下载RKE

最新rke 0.0.9 但是我一直跑不起来，用0.0.8可以

```bash
wget -O rke https://github.com/rancher/rke/releases/download/v0.0.8-dev/rke_linux-amd64

chmod +x ./rke

wget https://raw.githubusercontent.com/rancher/rke/master/cluster.yml
```



## 开启ssh-key

如果未开启ssh-key登陆，会报 

![](http://ww1.sinaimg.cn/large/afaffa71ly1fn6p3zgd3ej20kz02474n.jpg)

### client 端

如果未安装 ssh-server，则通过 `sudo apt-get install openssh-server -y`安装

```bash
cd ~/.ssh
ssh-keygen -t rsa -b 2048
scp  ~/.ssh/id_rsa.pub server_name@server_ip:/path/to/rsa/key/id_rsa.pub
```

## server 端

```bash
mkdir ~/.ssh/
cat /path/to/rsa/key/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/*
rm -rf /path/to/rsa/key/id_rsa.pub
sudo service ssh restart
```

参考 [SSH公钥(public key)验证](http://www.cnblogs.com/xiongmaolinux/p/5345125.html)



## 禁用swap

[Failed to deploy addon execute job: Failed to get job complete status: <nil>](https://github.com/rancher/rke/issues/130)

```bash
$ docker logs kubelet

I1212 01:50:20.857798   19652 server.go:422] --cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /
error: failed to run Kubelet: Running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename				Type		Size	Used	Priority /dev/sda5                               partition	1046524	2000	-1]
```

```bash
sudo swapoff -a
```



参考 [升级到Kubernetes1.8.4的配置细节差异以及k8s几个不常见的坑](http://blog.csdn.net/cleverfoxloving/article/details/78424293?locationNum=2&fps=1)



## 运行

cluster.yml

```yaml
---
auth:
  strategy: x509

network:
  plugin: flannel

ssh_key_path: /home/root/.ssh/id_rsa

nodes:
  - address: 172.60.20.12
    user: root
    role: [controlplane, etcd]
  - address: 172.60.20.13
    user: root
    role: [worker]

services:
  etcd:
    image: quay.io/coreos/etcd:latest
  kube-api:
    image: rancher/k8s:v1.8.3-rancher2
    service_cluster_ip_range: 10.233.0.0/18
    extra_args:
      v: 4
  kube-controller:
    image: rancher/k8s:v1.8.3-rancher2
    cluster_cidr: 10.233.64.0/18
    service_cluster_ip_range: 10.233.0.0/18
  scheduler:
    image: rancher/k8s:v1.8.3-rancher2
  kubelet:
    image: rancher/k8s:v1.8.3-rancher2
    cluster_domain: cluster.local
    cluster_dns_server: 10.233.0.3
    infra_container_image: anjia0532/pause-amd64:3.0
  kubeproxy:
    image: rancher/k8s:v1.8.3-rancher2

system_images:
  alpine: alpine:latest
  nginx_proxy: rancher/rke-nginx-proxy:0.1.0
  cert_downloader: rancher/rke-cert-deployer:0.1.0
  kubedns_image: anjia0532/k8s-dns-kube-dns-amd64:1.14.5
  dnsmasq_image: anjia0532/k8s-dns-dnsmasq-nanny-amd64:1.14.5
  kubedns_sidecar_image: anjia0532/k8s-dns-sidecar-amd64:1.14.5
  kubedns_autoscaler_image: anjia0532/cluster-proportional-autoscaler-amd64:1.0.0

addons: |-
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-nginx
      namespace: default
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80

```

```bash
./rke up --config ./cluster.yml
```

```
INFO[0041] [addons] Saving addon ConfigMap to Kubernetes 
INFO[0041] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-network-plugin 
INFO[0041] [addons] Executing deploy job..              
INFO[0046] [addons] Setting up KubeDNS                  
INFO[0046] [addons] Saving addon ConfigMap to Kubernetes 
INFO[0046] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-kubedns-addon 
INFO[0046] [addons] Executing deploy job..              
INFO[0051] [addons] KubeDNS deployed successfully..     
INFO[0051] [addons] Setting up user addons..            
INFO[0051] [addons] Saving addon ConfigMap to Kubernetes 
INFO[0051] [addons] Successfully Saved addon to Kubernetes ConfigMap: rke-user-addon 
INFO[0051] [addons] Executing deploy job..              
INFO[0056] [addons] User addon deployed successfully..  
INFO[0056] Finished building Kubernetes cluster successfully 
```



```bash
$ docker ps
CONTAINER ID        IMAGE                                                                                                                     COMMAND                  CREATED             STATUS              PORTS                              NAMES
437cde03416b        anjia0532/k8s-dns-sidecar-amd64@sha256:6f49768b598f9b74ee3774c19406a1512567e73f103544cf6bd3e04420ed669a                   "/sidecar --v=2 --..."   37 minutes ago      Up 37 minutes                                          k8s_sidecar_kube-dns-66fc7d58bf-lpps5_kube-system_c9c4615e-f288-11e7-8429-000c29218fe1_0
08db9a075ecd        nginx@sha256:926b086e1234b6ae9a11589c4cece66b267890d24d1da388c96dd8795b2ffcfb                                             "nginx -g 'daemon ..."   37 minutes ago      Up 37 minutes                                          k8s_my-nginx_my-nginx_default_ccc273f2-f288-11e7-8429-000c29218fe1_0
05e25488ec1a        anjia0532/k8s-dns-dnsmasq-nanny-amd64@sha256:667c279741b1efe1e667dabc022f04f04ff0d1d35af934c0f2508bee903c6c23             "/dnsmasq-nanny -v..."   37 minutes ago      Up 37 minutes                                          k8s_dnsmasq_kube-dns-66fc7d58bf-lpps5_kube-system_c9c4615e-f288-11e7-8429-000c29218fe1_0
e7064c9f4738        anjia0532/cluster-proportional-autoscaler-amd64@sha256:03795e1fbcc5ad4071ec969012c60dc53c8dce1b542c94701164b1224c53abaf   "/cluster-proporti..."   37 minutes ago      Up 37 minutes                                          k8s_autoscaler_kube-dns-autoscaler-84476ff9c8-57sf9_kube-system_c9b608ef-f288-11e7-8429-000c29218fe1_0
4b845893a51f        anjia0532/pause-amd64:3.0                                                                                                 "/pause"                 38 minutes ago      Up 38 minutes                                          k8s_POD_my-nginx_default_ccc273f2-f288-11e7-8429-000c29218fe1_0
c2ded5b27a95        anjia0532/k8s-dns-kube-dns-amd64@sha256:d965a1a9b53b254e2bedcbf86d0eba9378ea47084771e20f744cfbf7a1025ba6                  "/kube-dns --domai..."   38 minutes ago      Up 38 minutes                                          k8s_kubedns_kube-dns-66fc7d58bf-lpps5_kube-system_c9c4615e-f288-11e7-8429-000c29218fe1_0
97736e09c32b        anjia0532/pause-amd64:3.0                                                                                                 "/pause"                 38 minutes ago      Up 38 minutes                                          k8s_POD_kube-dns-66fc7d58bf-lpps5_kube-system_c9c4615e-f288-11e7-8429-000c29218fe1_0
1f16ce846485        anjia0532/pause-amd64:3.0                                                                                                 "/pause"                 38 minutes ago      Up 38 minutes                                          k8s_POD_kube-dns-autoscaler-84476ff9c8-57sf9_kube-system_c9b608ef-f288-11e7-8429-000c29218fe1_0
df03a750637d        quay.io/coreos/flannel-cni@sha256:77bf1017845afb65e2603d8573e9a2d649eb645a4f7fe4843f17e276b8126968                        "/install-cni.sh"        38 minutes ago      Up 38 minutes                                          k8s_install-cni_kube-flannel-6kff4_kube-system_c78facb9-f288-11e7-8429-000c29218fe1_0
e4d8b49b1f63        quay.io/coreos/flannel@sha256:60d77552f4ebb6ed4f0562876c6e2e0b0e0ab873cb01808f23f55c8adabd1f59                            "/opt/bin/flanneld..."   38 minutes ago      Up 38 minutes                                          k8s_kube-flannel_kube-flannel-6kff4_kube-system_c78facb9-f288-11e7-8429-000c29218fe1_0
90b858291c7f        anjia0532/pause-amd64:3.0                                                                                                 "/pause"                 38 minutes ago      Up 38 minutes                                          k8s_POD_kube-flannel-6kff4_kube-system_c78facb9-f288-11e7-8429-000c29218fe1_0
fd1fd7b1928c        rancher/k8s:v1.8.3-rancher2                                                                                               "kube-proxy --v=2 ..."   39 minutes ago      Up 39 minutes                                          kube-proxy
1d1d03d6de8c        rancher/k8s:v1.8.3-rancher2                                                                                               "kubelet --v=2 --a..."   39 minutes ago      Up 39 minutes                                          kubelet
c8f19935efd4        rancher/k8s:v1.8.3-rancher2                                                                                               "kube-scheduler --..."   39 minutes ago      Up 39 minutes                                          scheduler
1d234c5c8247        rancher/k8s:v1.8.3-rancher2                                                                                               "kube-controller-m..."   39 minutes ago      Up 39 minutes                                          kube-controller
c3eb4391249f        rancher/k8s:v1.8.3-rancher2                                                                                               "kube-apiserver --..."   39 minutes ago      Up 39 minutes                                          kube-api
8ff019fee9d8        quay.io/coreos/etcd:latest                                                                                                "/usr/local/bin/et..."   39 minutes ago      Up 39 minutes       0.0.0.0:2379-2380->2379-2380/tcp   etcd

```

```bash
$ docker images
REPOSITORY                                        TAG                 IMAGE ID            CREATED             SIZE
quay.io/coreos/etcd                               latest              30d9f8842f26        3 days ago          37.2 MB
nginx                                             latest              3f8a4339aadd        10 days ago         108 MB
rancher/rke-service-sidekick                      0.1.0               0f60235e607e        3 weeks ago         746 B
alpine                                            latest              e21c333399e0        5 weeks ago         4.14 MB
rancher/rke-cert-deployer                         0.1.0               c8907e804cfe        5 weeks ago         9.09 MB
rancher/k8s                                       v1.8.3-rancher2     bbbe40353d71        7 weeks ago         1.54 GB
quay.io/coreos/flannel                            v0.9.1              2b736d06ca4c        7 weeks ago         51.3 MB
anjia0532/k8s-dns-sidecar-amd64                   1.14.5              fed89e8b4248        3 months ago        41.8 MB
anjia0532/k8s-dns-kube-dns-amd64                  1.14.5              512cd7425a73        3 months ago        49.4 MB
anjia0532/k8s-dns-dnsmasq-nanny-amd64             1.14.5              459944ce8cc4        3 months ago        41.4 MB
quay.io/coreos/flannel-cni                        v0.2.0              7252edf978c0        4 months ago        49.8 MB
anjia0532/cluster-proportional-autoscaler-amd64   1.0.0               e183460c484d        14 months ago       48.2 MB
anjia0532/pause-amd64                             3.0                 99e59f495ffa        20 months ago       747 kB
```

参考 

- [RKE快速上手指南：开源的轻量级K8S安装程序](https://www.cnrancher.com/an-introduction-to-rke/)
- [rancher/rke#README.md](https://github.com/rancher/rke/blob/master/README.md)
- [An Introduction to Rancher Kubernetes Engine (RKE)](https://rancher.com/an-introduction-to-rke/)

