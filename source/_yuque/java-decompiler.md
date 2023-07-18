---
title: 016-JDK8+可用的反编译工具(JD_GUI+Procyon)
urlname: java-decompiler
date: '2019-04-18 12:10:00 +0800'
tags:
  - java
  - jdk8
  - 反编译
categories: java
---

> 这是坚持技术写作计划（含翻译）的第 16 篇，定个小目标 999，每周最少 2 篇。

本文是源于一次逆向 android app，辛苦脱壳后得到 `classes_dumped_29-dex2jar.jar` ，要得到源码，但是又不想降级 jdk 到 1.7 来迁就 jd_gui。花了一分钟，找到 jd_gui 在 1.8 下的用法,至于 基于 procyon 的 UI luyten 纯是凑数。

<!-- more -->

## JD_GUI

打开  [http://java-decompiler.github.io/](http://java-decompiler.github.io/)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1555556972336-dbd69397-f0ee-454a-b39c-779eca71dd63.png#align=left&display=inline&height=678&originHeight=678&originWidth=1219&size=90920&status=done&width=1219)
其实官网已经很明显了，大家之所以以讹传讹，认为 JD_GUI 不支持 1.8，大多是被度娘或者 CSDN 荼毒。
1.4.0 及以前的 jd_gui，在 1.8 打开一般是
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1555557122787-0a3fac4f-3f1f-4da0-a48c-d7547ab217fa.png#align=left&display=inline&height=151&originHeight=151&originWidth=357&size=5587&status=done&width=357)

下载并解压预览版，然后 `java -jar jd-gui-1.4.1.jar` 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1555557322830-cb8915d1-e7bd-4e98-921c-ede3571d0a02.png#align=left&display=inline&height=505&originHeight=505&originWidth=1094&size=76642&status=done&width=1094)
熟悉的界面，熟悉的配方。

### 官方截图

![LocalMethodClasses.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1555557366601-a7591bd1-09c5-4287-833c-fc0324d02ee4.png#align=left&display=inline&height=425&originHeight=665&originWidth=1168&size=88222&status=done&width=746)![TryWithResources.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1555557367229-70fe5c9d-b2be-421a-83df-0d90e890899c.png#align=left&display=inline&height=426&originHeight=665&originWidth=1165&size=96487&status=done&width=746)![DefaultMethodsInInterface.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1555557367292-f0ef5b46-5a2f-4576-9182-203baae7a0ff.png#align=left&display=inline&height=426&originHeight=665&originWidth=1165&size=88222&status=done&width=746)![Lambda.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1555557367324-1e872a6b-a4e2-4ce7-a852-305362ae113f.png#align=left&display=inline&height=426&originHeight=665&originWidth=1165&size=82190&status=done&width=746)

## procyon + luyten

下载最新版的 [luyten.jar](https://github.com/deathmarine/Luyten/releases) ,然后    `java -jar luyten-0.5.4.jar` 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1555557560115-d10ea3a0-3d78-46e7-a6ab-13047ae1f347.png#align=left&display=inline&height=516&originHeight=516&originWidth=942&size=68141&status=done&width=942)
只是轻度使用的话，两个差不多，建议用 jd_gui,起码搜索速度能甩 luyten 10 条街啊。

## 结语

是不是以为会有类似 lambda 反编译比对一类的评测文？答案是，你想多了。这些工具只要有数就行，一个不好用，换另一个就行。

其实，一般情况下，使用独立反编译工具的可能性很小,一般是 IDE 的插件居多，比如，[cnfree/Eclipse-Class-Decompiler](https://github.com/cnfree/Eclipse-Class-Decompiler) ,而 idea 默认有简易版的反编译插件。足以应付日常工作中零星的反编译用途。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
