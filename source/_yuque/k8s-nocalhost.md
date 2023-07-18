---
title: 077-加快云原生应用开发速度(Nocalhost篇)
urlname: k8s-nocalhost
date: '2022-08-19 19:35:21 +0800'
tags:
  - nocalhost
  - k8s
  - 云原生
  - docker
  - 微服务
categories:
  - k8s
---

> 这是坚持技术写作计划（含翻译）的第 77 篇，定个小目标 999，每周最少 2 篇。

本文主要以 Nocalhost 工具为例，讲解如何快速开发调试跑在 k8s 集群的微服务等云原生应用。

<!-- more -->

## 安装 Nocalhost

Nocalhost 支持 [VScode](https://nocalhost.dev/zh-CN/docs/installation#%E5%AE%89%E8%A3%85-vs-code-%E6%8F%92%E4%BB%B6) 和 [Jetbrains](https://nocalhost.dev/zh-CN/docs/installation#%E5%AE%89%E8%A3%85-jetbrains-%E6%8F%92%E4%BB%B6) (Jetbrains 插件目前不支持 `2022.2.*`,提了[PR](https://github.com/nocalhost/nocalhost-intellij-plugin/pull/180) 没人合，感觉 idea plugin 已经没人维护了,我 Fork 了一个，使用 Github Action 构建了一个 `2022.2.*` 可用的版本 [https://github.com/anjia0532/nocalhost-intellij-plugin/actions/runs/2831564805](https://github.com/anjia0532/nocalhost-intellij-plugin/actions/runs/2831564805))

## 添加 K8S 集群

可以直接复制 kubeconfig 文本粘贴，也可以下载 kubeconfig 文件到本地

详见 [集群管理](https://nocalhost.dev/zh-CN/docs/guides/manage-cluster)

## 创建实例应用

官方是用 nocalhost-server 或者 istio 的 bookinfo 为例。

此处以 Java 系常用的 Spring Boot 为例。

### 基于 Spring Initializr 创建 demo 应用

![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660903078040-d049add0-a357-4997-b87a-68c32df9132a.png#clientId=ue96590a1-5f30-4&from=paste&height=632&id=u80188ff8&originHeight=632&originWidth=781&originalType=binary∶=1&rotation=0&showTitle=false&size=56725&status=done&style=none&taskId=u7f8ab72f-990e-4873-8ec0-54c8030d288&title=&width=781)
在 `com.example.nocalhostspringdemo` 包下创建 Controller.java 类

```java
package com.example.nocalhostspringdemo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author AnJia
 * @since 2022-08-19 16:47
 */
@RestController("/")
public class Controller {
    @GetMapping("/nocalhost-demo")
    public String test() {
        return "hello nocalhost";
    }
}

```

### 构建 jar 包

![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660904921387-3dc21202-461d-4213-9f56-cb7c0ad34dd7.png#clientId=ue96590a1-5f30-4&from=paste&height=427&id=u39a75560&originHeight=427&originWidth=874&originalType=binary∶=1&rotation=0&showTitle=false&size=285734&status=done&style=none&taskId=u00181554-fdb6-4964-b301-95e1e88404f&title=&width=874)

## 部署应用到 K8S 集群

将下列 yaml 代码保存为 `nocalhost-test.yaml` ,执行 `kubectl apply -f ./nocalhost-test.yaml`

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: nocalhost-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-demo
  namespace: nocalhost-test
  labels:
    app: spring-boot-demo
spec:
  replicas: 3
  template:
    metadata:
      name: spring-boot-demo
      labels:
        app: spring-boot-demo
    spec:
      containers:
        - name: spring-boot-demo
          image: anjia0532/openjdk-8-alpine-lib:3.5.2
          imagePullPolicy: IfNotPresent
          command:
            - tail
            - -f
            - /dev/null
      restartPolicy: Always
  selector:
    matchLabels:
      app: spring-boot-demo

```

也可以通过 helm 进行安装

## 使用 Nocalhost 部署调试服务

在项目根目录创建个 `.nocalhost` 文件夹，并将下列代码保存为 `config.yaml`

```yaml
name: "spring-boot-demo"
serviceType: "deployment"
containers:
  - name: "spring-boot-demo"
    hub: null
    dev:
      gitUrl: ""
      image: "anjia0532/openjdk-8-alpine-lib:3.5.2"
      shell: "sh"
      workDir: ""
      storageClass: ""
      resources: null
      persistentVolumeDirs: []
      command:
        debug:
          - java
          - -jar
          - -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
          - /home/nocalhost-dev/target/nocalhost-spring-demo-0.0.1-SNAPSHOT.jar
        run:
          - java
          - -jar
          - /home/nocalhost-dev/target/nocalhost-spring-demo-0.0.1-SNAPSHOT.jar
      debug:
        language: "java"
        remoteDebugPort: 5005
      hotReload: true
      sync:
        type: "sendReceive"
        mode: "gitIgnore"
      env: []
      portForward: []
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660903844217-d1b8a7e8-695d-4a80-a458-af8bf9697054.png#clientId=ue96590a1-5f30-4&from=paste&height=564&id=u2d9860ba&originHeight=564&originWidth=517&originalType=binary∶=1&rotation=0&showTitle=false&size=135520&status=done&style=none&taskId=ubf73d9a9-f5d1-404d-851c-465b0a07a23&title=&width=517)
在 Controller 类里加上断点，使用远程 debug 模式启动应用。然后把 8080 端口转到本地。

本地浏览器访问 [http://localhost:8080/nocalhost-demo](http://localhost:8080/nocalhost-demo) 命中断点，意味着可以进行断点调试。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660905132372-799e91c6-d419-4c67-867b-e071ddfce63a.png#clientId=ue96590a1-5f30-4&from=paste&height=596&id=u36e929fe&originHeight=596&originWidth=1274&originalType=binary∶=1&rotation=0&showTitle=false&size=479444&status=done&style=none&taskId=u84a9180d-2b28-4e9c-807b-e4daa70ea7b&title=&width=1274)

参考 [Nocalhost Jetbrains Debug](https://nocalhost.dev/zh-CN/docs/guides/debug/jetbrains-debug)

可以用于预发布环境下，进行调试，省去了 本地开发，提交代码，流水线构建推送镜像，发版，切换流量，看日志 的过程，生产不建议这么用，有些时候会发布失败。

另外 java 系可以结合 springboot 的热加载或者 jrebel 的热部署功能，实现修改后，不用重启，自动生效的效果。

其余 Lua，JS，Python，Golang 等也都可以使用本方法。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/08/19/k8s-nocalhost/)
- [我的掘金](https://juejin.cn/post/7133534655864635423/)
