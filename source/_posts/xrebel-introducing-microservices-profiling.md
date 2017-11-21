---
title: 微服务场景下性能问题排查神器之xrebel
date: 2017-11-21 14:57:14
tags: [jrebel,xrebel,micro-service,spring-cloud]
---

对于java应用性能跟踪其实有很多种手段，本文只是针对`xrebel`的使用做一些简单讲解（`单体应用`和`微服务应用`）。分布式跟踪有很多，比如zipkin等，详见 [分布式跟踪系统（一）：Zipkin的背景和设计][]，但是太重了，不适合小规模团队，开发时期用。

而且以zipkin为例，仅仅是A服务调用B服务耗时多少，并不会显示详细的线程，堆栈信息。需要搭配其他手段进行排查。

示例：

![](http://ww1.sinaimg.cn/large/afaffa71ly1flpr6fphdbj21h10q30xi.jpg)

<!--more-->

## [下载xrebel][]

目前最新版本 [xrebel-3.4.1.zip][]

## [xrebel 支持的框架及场景][Xrebel支持的框架及场景]


## [快速安装][]

xrebel支持eclipse和idea,同时有eclipse插件，建议使用独立方式安装。

1. 下载xrebel.zip 并解压到本地，e.g. `D:\xrebel`
2. 在tomcat也好，idea,eclipse也好，修改vm 参数，添加 `-javaagent:[path/to/xrebel]/xrebel.jar`

下面分别是idea,eclipse

![](http://ww1.sinaimg.cn/large/afaffa71ly1flpqqhxvfhj20jd0moaat.jpg)

![](http://ww1.sinaimg.cn/large/afaffa71ly1flpqqhyux0j20wq0spmz7.jpg)

默认是可以试用14天的，建议支持正版，毕竟大家都是吃这行饭的。而且 [xrebel][] [jrebel][] [jrebel for android][JrebelForAndroid] 给你省的时间，绝对值这个价。 jrebel有个免费的社区计划 [https://my.jrebel.com/][]

## web ui

打开 web 服务页面，xrebel会直接注入到你的页面中，左下角会出现 `xrebel`的`toolbar`，(e.g. http://localhost:8080)

![](http://ww1.sinaimg.cn/large/afaffa71ly1flpqswphn4j20220bqjrc.jpg)

或者通过 访问`服务/xrebel` (e.g. http://localhost:8080/xrebel) 打开单独页面，适用于webservice,restful 等无页面场景

![](http://ww1.sinaimg.cn/large/afaffa71ly1flpqvgh2muj20md0f23z9.jpg)

如果不想注入到页面中，只想通过`服务/xrebel`访问，则可以添加 `-Dxrebel.injection=true|false` ，默认为`true`

其余开关参数 参见  [XRebel launch parameters][XrebelLaunchParameters]


## xrebel 简单使用教程

参考 [Using XRebel][UsingXrebel]

![](http://ww1.sinaimg.cn/large/afaffa71ly1flprxem5gdj20tq0h3tc1.jpg)
![](http://ww1.sinaimg.cn/large/afaffa71ly1flprxekmsfj20wx0h3jua.jpg)
![](http://ww1.sinaimg.cn/large/afaffa71ly1flprxeljmhj20tt0h3wg0.jpg)
![](http://ww1.sinaimg.cn/large/afaffa71ly1flprxelpxkj20th0h10vd.jpg)

## 微服务

参考 [Microservices][] 和 [XRebel 3.0: introducing microservices profiling][Xrebel3.0:IntroducingMicroservices] 

确保调用方，和被调用方，都开了xrebel，

效果如下

![](http://ww1.sinaimg.cn/large/afaffa71ly1flpr6fphdbj21h10q30xi.jpg)


## 启用xrebel调试

参考 [Debugging with XRebel enabled][DebuggingWithXrebelEnabled]


## 题外话 静态资源分离的必要性

为嘛建议将静态文件分离？通过xrebel就可以清晰看出来

![](http://ww1.sinaimg.cn/large/afaffa71ly1flprzm06cej20lo09iq3n.jpg)


博客 [https://anjia.ml/2017/11/21/xrebel-introducing-microservices-profiling/][blog]
掘金 [https://juejin.im/post/5a13e3db6fb9a045186a5bfc][juejin]
简书 [http://www.jianshu.com/p/0029c32dde4e][jianshu]


[blog]: https://anjia.ml/2017/11/21/xrebel-introducing-microservices-profiling/
[juejin]: https://juejin.im/post/5a13e3db6fb9a045186a5bfc
[jianshu]: http://www.jianshu.com/p/0029c32dde4e
[分布式跟踪系统（一）：Zipkin的背景和设计]: http://manzhizhen.iteye.com/blog/2348175
[下载xrebel]: https://zeroturnaround.com/software/xrebel/download/
[xrebel-3.4.1.zip]: https://zeroturnaround.com/software/xrebel/download/thank-you/?file=xrebel/releases/xrebel-3.4.1.zip
[Xrebel支持的框架及场景]: http://manuals.zeroturnaround.com/xrebel/support/index.html#
[快速安装]: https://zeroturnaround.com/software/xrebel/quick-start/
[XrebelLaunchParameters]: http://manuals.zeroturnaround.com/xrebel/use/advanced.html#xrebel-launch-parameters
[Microservices]: http://manuals.zeroturnaround.com/xrebel/use/advanced.html#microservices
[Xrebel3.0:IntroducingMicroservices]: https://zeroturnaround.com/rebellabs/xrebel-3-0-introducing-microservices-profiling/
[UsingXrebel]: http://manuals.zeroturnaround.com/xrebel/use/index.html#
[jrebel]: https://zeroturnaround.com/software/jrebel/
[JrebelForAndroid]: https://zeroturnaround.com/software/jrebel-for-android/
[xrebel]: https://zeroturnaround.com/software/xrebel/
[https://my.jrebel.com/]: https://my.jrebel.com/
[DebuggingWithXrebelEnabled]: http://manuals.zeroturnaround.com/xrebel/use/advanced.html#debugging-with-xrebel-enabled
