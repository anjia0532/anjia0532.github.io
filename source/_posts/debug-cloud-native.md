---
title: 云原生时代如何方便的进行本地调试
urlname: debug-cloud-native
date: 2019-01-21 12:00:00 +0800
tags: [k8s,云原生,devops,微服务,容器,jrebel,telepresence]
categories: [k,8,s]
---

云原生的四要素：持续交付、DevOps、微服务、容器，虽然极大的解放了生产力，但是不可避免的也带来了诸多问题，本文不做延伸，感兴趣的，可以自行百度。
本文只为解决微服务(本文以 Spring Cloud 为例)+Kubernetes 开发调试低效问题。

![07e15debbba7479aabef8f861f3ef5f4.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/226273/1548032278508-325e41b8-29f1-46de-b1a0-5a97e3de41e8.jpeg#align=left&display=inline&height=486&name=07e15debbba7479aabef8f861f3ef5f4.jpg&originHeight=704&originWidth=1080&size=72220&width=746)

<!-- more -->

## [telepresence](https://www.telepresence.io/)

如果团队内成员都有 k8s 基础，并且都用 win10 或者 linux,macos，那建议直接用 telepresence,简单直接。详见  [Fast development workflow with Docker and Kubernetes](https://www.telepresence.io/tutorials/docker)，[A development workflow for Kubernetes services](https://articles.microservices.com/a-development-workflow-for-kubernetes-services-10ee017d752a)

## Service 映射

如果团队内 k8s 基础弱，或者硬件条件不满足，可以使用 Service 映射方案，在 k8s 集群里创建一个 Service 和 Endpoint，然后进行绑定。但是适用于单向的，比如，k8s 访问外部 mysql，如果要逆向访问，不好意思，不支持。

## 静态路由

[https://github.com/jkwong888/k8s-add-static-routes](https://github.com/jkwong888/k8s-add-static-routes)

## TDD

如果团队对于单院测试和 Mock 掌握的比较好，可以直接开启 TDD 模式，省事省心

## 远程调试

k8s 集群暴露远程调试接口。[Remote debugging Spring Boot on Kubernetes](https://itnext.io/remote-debugging-spring-boot-on-kubernetes-a5f96a40e5c0)

## 开发机纳入集群

应用发到本地 pod 里，省的走 cicd 那么费劲了

## 热部署

开发机纳入集群后，把 target\class 挂载到本地卷，并且配置上 rebel.xml，idea build 后生成 class，然后 pod 里触发 jrebel 的热部署。 参考  [https://www.telepresence.io/tutorials/java#hot-code-replace](https://www.telepresence.io/tutorials/java#hot-code-replace)
