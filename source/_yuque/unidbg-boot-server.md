---
title: 066-unidbg-boot-server零基础入门
urlname: unidbg-boot-server
date: '2021-11-01 19:35:21 +0800'
tags:
  - 逆向
  - unidbg
categories:
  - 逆向
---

> 这是坚持技术写作计划（含翻译）的第 66 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解 [anjia0532/unidbg-boot-server](https://github.com/anjia0532/unidbg-boot-server) 如何使用，以及一些群友反馈的常见问题的处理。

<!-- more -->

## 啥是 unidbg

简单来说 [zhkl0228/unidbg](https://github.com/zhkl0228/unidbg) 是[凯神](https://github.com/zhkl0228) 的基于[unicorn-engine/unicorn](https://github.com/unicorn-engine/unicorn) 开发的一个可编程的 Android 和 ios 的模拟器。当然做好心理准备，unidbg 并不是一个商业完备的模拟器，随着深入使用，几乎免不了的会遇到各种问题（比如内存管理，比如某些特性缺失啥的），但是，又不是不能用，要啥自行车
![](https://cdn.nlark.com/yuque/0/2021/jpeg/226273/1635737443232-e4233df4-7449-4c92-916b-f12401ef348a.jpeg#clientId=uc5ce4b29-69a0-4&from=paste&id=ud7e3a4e1&originHeight=314&originWidth=500&originalType=url∶=1&status=done&style=none&taskId=u183c367b-3c23-4d68-b82d-cf4e61375ea)

## unidbg-boot-server 和 unidbg 的关系

就像 [zhkl0228/unidbg](https://github.com/zhkl0228/unidbg) 之于[unicorn-engine/unicorn](https://github.com/unicorn-engine/unicorn)，[anjia0532/unidbg-boot-server](https://github.com/anjia0532/unidbg-boot-server) 虽然最初定位是一个开箱机用的 unidbg 的高性能多线程的 http server，但是只是这点的话，那格局就小了，目前 unidbg-boot-server 的定位是 基于 unidbg 的开箱机用，新手友好，集成最佳实践的 unidbg 脚手架，Javaer 可以无门槛上手，pythoner 等无 java 基础的也可以低成本上手。

## 环境准备

1. [Idea](https://www.jetbrains.com/idea/download/)，以及一些简单的 jetbrains 家工具的使用经验(idea,goland,pycharm,webstorm 等)
2. [Java8](https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html) mac 或者 linux 也可以用 openjdk,但是别自以为是的用 java11+,除非你确定你能 hold 住。Windows 建议参考我的 [jdk 绿色免安装](https://anjia0532.github.io/2017/05/17/jdk-zip/) ，虽然麻烦点，但是可以避免一些坑。如果是 linux 或者 mac，想用 oracle 的 jdk，可以参考 [044-wget 免登陆下载 jdk 8u291](https://anjia0532.github.io/2019/09/18/wget-jdk-8u221/)
3. [Maven3.5](https://maven.apache.org/download.cgi)以上 ，如果电脑没有安装 Maven，最简单办法是将下面的 `mvn` 命令替换成 `mvnw`(如果是 linux/mac 一类的，要替换成`./mvnw`,并且先执行`chmod +x ./mvnw`) ，会自动下载 maven
4. [git](https://git-scm.com/downloads) 建议最新版
5. idea [Lombok 插件](https://plugins.jetbrains.com/plugin/6317-lombok)
6. 【可选】配置 maven 加速器 [阿里云加速器](https://developer.aliyun.com/mvn/guide)，博客园相关文章 [aliyun 阿里云 Maven 仓库地址——加速你的 maven 构建](https://www.cnblogs.com/geektown/p/5705405.html)
7. 【可选】掌握科学上网或者加速访问 github 的方法，参考 [提高国内访问 github 速度的 9 种方法！](https://segmentfault.com/a/1190000038298623)，稳定可靠的是使用 gitee 转一层，长期可用。简单省事的是使用反代服务器，比如 [https://hub.fastgit.org/](https://hub.fastgit.org/)，一般是将`github.com` 替换成 `hub.fastgit.org`，但是随着用的人多，会被拦截，提示被滥用。

## 下载 unidbg-boot-server 并运行

### 快速体验

使用命令行

```bash
# 也可以试试hub.fastgit.org 加速器，不保证长期可用
# git clone https://hub.fastgit.org/anjia0532/unidbg-boot-server.git
git clone https://github.com/anjia0532/unidbg-boot-server.git

# 体验jar版本,打成jar包，-T是10线程，加速构建，-DskipTests 是跳过单元测试
mvn package -T10 -DskipTests

# 没有maven就用 mvnw package -T10 -DskipTests (linux等需要用 chmod +x ./mvnw && ./mvnw package -T10 -DskipTests)

# 运行
java -jar target\unidbg-boot-server-0.0.1-SNAPSHOT.jar
```

### 使用 idea

参考 [在 IDEA 中配置及使用 Maven 的全过程](https://zhuanlan.zhihu.com/p/122429605) 配置 maven
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1635739233579-e8268fb6-a65b-4e4c-a717-162200678eb2.png#clientId=uc5ce4b29-69a0-4&from=paste&height=675&id=ufde6ac76&originHeight=675&originWidth=1794&originalType=binary∶=1&size=912779&status=done&style=none&taskId=u955b69b8-51ce-4202-90f5-6a40a517e25&width=1794)

### 运行

```bash
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.5.5)

2021-11-01 12:01:17.689  INFO 33648 --- [           main] c.a.u.UnidbgServerApplication            : Starting UnidbgServerApplication using Java 1.8.0_221 on AnJia with PID 33648 (D:\AnJia\Work\workspace\idea\unidbg-boot-server1\target\classes started by AnJia in D:\AnJia\Work\workspace\idea\unidbg-boot-server1)
2021-11-01 12:01:17.692  INFO 33648 --- [           main] c.a.u.UnidbgServerApplication            : The following profiles are active: dev
2021-11-01 12:01:19.224  INFO 33648 --- [           main] c.a.u.service.TTEncryptServiceWorker     : 线程池为:8
2021-11-01 12:01:19.225  INFO 33648 --- [435772@424de326] c.a.u.service.TTEncryptServiceWorker     : 是否启用动态引擎:true,是否打印详细信息:false
2021-11-01 12:01:19.594  WARN 33648 --- [           main] ion$DefaultTemplateResolverConfiguration : Cannot find template location: classpath:/templates/ (please add some templates or check your Thymeleaf configuration)
2021-11-01 12:01:19.699  INFO 33648 --- [           main] c.a.u.UnidbgServerApplication            : Started UnidbgServerApplication in 2.755 seconds (JVM running for 4.456)
2021-11-01 12:01:19.710  INFO 33648 --- [           main] c.a.u.UnidbgServerApplication            :
----------------------------------------------------------
	应用: 		unidbg-boot-server 已启动!
	地址: 		http://127.0.0.1:9999/
	演示访问: 	curl http://127.0.0.1:9999/api/tt-encrypt/encrypt (linux)
	演示访问: 	http://127.0.0.1:9999/api/tt-encrypt/encrypt (windows: 浏览器直接打开)
	常见问题: 	https://github.com/anjia0532/unidbg-boot-server/blob/main/QA.md
	配置文件: 	[application, application-dev]
----------------------------------------------------------
2021-11-01 12:01:19.710  INFO 33648 --- [           main] c.a.u.UnidbgServerApplication            :
----------------------------------------------------------

2021-11-01 12:01:20.201  INFO 33648 --- [435772@424de326] c.a.u.service.TTEncryptServiceWorker     : 是否启用动态引擎:true,是否打印详细信息:false
2021-11-01 12:01:20.327  INFO 33648 --- [435772@424de326] c.a.u.service.TTEncryptServiceWorker     : 是否启用动态引擎:true,是否打印详细信息:false
2021-11-01 12:01:20.456  INFO 33648 --- [435772@424de326] c.a.u.service.TTEncryptServiceWorker     : 是否启用动态引擎:true,是否打印详细信息:false
2021-11-01 12:01:20.556  INFO 33648 --- [435772@424de326] c.a.u.service.TTEncryptServiceWorker     : 是否启用动态引擎:true,是否打印详细信息:false
2021-11-01 12:01:20.635  INFO 33648 --- [435772@424de326] c.a.u.service.TTEncryptServiceWorker     : 是否启用动态引擎:true,是否打印详细信息:false
2021-11-01 12:01:20.725  INFO 33648 --- [435772@424de326] c.a.u.service.TTEncryptServiceWorker     : 是否启用动态引擎:true,是否打印详细信息:false
2021-11-01 12:01:20.816  INFO 33648 --- [435772@424de326] c.a.u.service.TTEncryptServiceWorker     : 是否启用动态引擎:true,是否打印详细信息:false

```

关键是这里,防止刚接触 spring boot 的，找不到访问地址，（主要是被问烦了），直接打印出来（就这样还 TM 有眼瞎看不到的，也是醉了）

```java
----------------------------------------------------------
	应用: 		unidbg-boot-server 已启动!
	地址: 		http://127.0.0.1:9999/
	演示访问: 	curl http://127.0.0.1:9999/api/tt-encrypt/encrypt (linux)
	演示访问: 	http://127.0.0.1:9999/api/tt-encrypt/encrypt (windows: 浏览器直接打开)
	常见问题: 	https://github.com/anjia0532/unidbg-boot-server/blob/main/QA.md
	配置文件: 	[application, application-dev]
----------------------------------------------------------
```

## 如何使用自己魔改的 unidbg

总有一些大佬发现标版 unidbg 满足不了（或者有 bug）自己，需要自行修改 unidbg。

```bash
# git clone https://hub.fastgit.org/zhkl0228/unidbg.git
git clone https://github.com/zhkl0228/unidbg.git

# 自己导入idea，各种修改后，改下pom.xml中的版本号

# 安装到本地
mvn clean install -Dgpg.skip=true -T10
```

## 如何自己写逻辑

运行 `src/test/java/com/anjia/unidbgserver/AutoGeneratorTest.java`
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1635747413764-bde4fde2-dd89-4bfc-9bfa-1efdd3fb11dd.png#clientId=uc5ce4b29-69a0-4&from=paste&height=719&id=u5655ac54&originHeight=719&originWidth=1735&originalType=binary∶=1&size=1564366&status=done&style=none&taskId=u6c912bc5-143d-4fe4-93fa-f1430570bea&width=1735)
把自己的业务逻辑移植/实现到 `src/main/java/com/anjia/unidbgserver/service/*Service.java`里

修改参数入参，修改 `src/main/java/com/anjia/unidbgserver/web/*Controller.java` 关于这个类里的注解不会用的，自行百度，或者参考
[Spring MVC @RequestMapping Annotation Example with Controller, Methods, Headers, Params, @RequestParam, @PathVariable](https://www.journaldev.com/3358/spring-requestmapping-requestparam-pathvariable-example)

主要 unidbg 模拟逻辑在 `com.anjia.unidbgserver.service.*Service` 里
`com.anjia.unidbgserver.web.*Controller` 是暴露给外部 http 调用的
`com.anjia.unidbgserver.service.*ServiceWorker` 是用多线程包装了一层业务逻辑
`com.anjia.unidbgserver.service.*ServiceTest` 是单元测试

## 可不可以不要单元测试/单元测试如何写/如何用

完全可以，但是不建议。
个人认为单元测试是这个项目的灵魂。想象一下原来写 unidbg 时，是不是得创建一个个逻辑类，自己写 N 多个 main 来测试不同的情况，还得配合着横七竖八的注释，来屏蔽代码，让它不生效。

用单元测试多优雅啊，先用 `src/test/java/com/anjia/unidbgserver/AutoGeneratorTest.java` 生成框架，直接在 Service 里写逻辑，遇到新项目还是研究阶段的，写不同的单元测试方法就行了。不用改了重启访问 url。直接调用方法不香吗？
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1635748120486-866913c9-c658-40a4-a74f-5cce4a1e4fff.png#clientId=uc5ce4b29-69a0-4&from=paste&height=478&id=ubc40daf0&originHeight=478&originWidth=1281&originalType=binary∶=1&size=689920&status=done&style=none&taskId=u35c96123-d401-40cf-86d4-bd5db109816&width=1281)

## 其他一些常见问题

都整理到这里了 [常见问题汇总](https://github.com/anjia0532/unidbg-boot-server/blob/main/QA.md) ，先看，先看，先看，别上来就问就问就问。

## [关于是否开源基于 Jnitrace 日志补环境代码说明](https://github.com/anjia0532/unidbg-boot-server/issues/1)

详见 [关于是否开源基于 Jnitrace 日志补环境代码说明](https://github.com/anjia0532/unidbg-boot-server/issues/1)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/11/01/unidbg-boot-server/)
- [我的掘金](https://juejin.cn/post/7025794546655035422/)
- [044-wget 免登陆下载 jdk 8u291](https://anjia0532.github.io/2019/09/18/wget-jdk-8u221/)
- [jdk 绿色免安装](https://anjia0532.github.io/2017/05/17/jdk-zip/)
