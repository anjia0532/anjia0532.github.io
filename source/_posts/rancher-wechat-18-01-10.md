---
title: 基于spring cloud的微服务实践
date: 2017-12-26 17:44:28
tags: [jhipster,spring,spring-boot,spring-cloud,微服务,microservices,service-mesh,k8s,kubernetes]
---

基于spring cloud的微服务实践
---

## 微服务概述

### 相关趋势图

从百度指数搜索 `微服务`、`spring boot`、`spring cloud`、`dubbo` 相关关键词，得到如下趋势（微服务的概念源于2014年3月Martin Fowler所写的一篇文章[Microservices](http://martinfowler.com/articles/microservices.html) ,所以选择从2014.03至今）

![](http://ww1.sinaimg.cn/large/afaffa71ly1fmtclday7oj20y309rgnr.jpg)

和`k8s`、`kubernetes` 、`docker` 的搜索趋势

![](http://ww1.sinaimg.cn/large/afaffa71ly1fmtcqtu6ejj20ye08bq3u.jpg)



### 微服务vs单体应用

单体应用好处

- 开发简单
- 容易测试
- 易于部署
- 事务回滚容易
- 无分布式管理，调用开销
- 重复功能/代码较少

单体应用缺点

- 迭代缓慢
- 维护困难
- 持续部署困难：微小改动，必须重启，不相干功能无法提供服务
- 牵一发而动全身：依赖项冲突，变更后，需要大量测试，防止影响其他功能
- 基础语言、框架升级缓慢
- 框架语言单一，无法灵活选用


微服务好处

- 敏捷性：按功能拆分，快速迭代
- 自主性：团队技术选型灵活(PHP,python,java,C#,nodejs,golang)，设计自主
- 可靠性：微服务故障只影响此服务消费者，而单体式应用会导致整个服务不可用
- 持续集成持续交付
- 可扩展性：热点功能容易扩展

微服务的缺点

- 性能降低：服务间通过网络调用
- 管理难度增大：增加了项目的复杂性
- 事务一致性

扩展阅读 [Introduction to Microservices](https://www.nginx.com/blog/introduction-to-microservices/)

### 框架选型

公司主要使用java，所以决定使用spring 框架中的`spring cloud`作为微服务基础框架，但是原生 spring cloud 学习曲线比较陡峭，需要学习`feign`,`zuul`,`eureka`,`hystrix`,`zipkin`,`ribbon`...

考虑到团队学习成本，故而采用了国外的开源框架 [jhipster官网](http://www.jhipster.tech/)  [jhipster github](https://github.com/jhipster)，登记在册的，使用jhipster的企业有217家

列举一下 [Technology stack](http://www.jhipster.tech/tech-stack/) 给的技术栈

#### 客户端技术栈

- angular4,5 or angularv1.x
- Bootstrap
- [HTML5 Boilerplate](http://html5boilerplate.com/)
- 兼容IE11+及现代浏览器
- 支持国际化
- 支持sass
- 支持spring websocket
- 支持yarn、bower管理js库
- 支持webpack、gulp.js构建，优化，应用
- 支持[Karma](http://karma-runner.github.io/), [Headless Chrome](https://github.com/GoogleChrome/puppeteer) 和 [Protractor](http://www.protractortest.org/) 进行前端单元测试测试
- 支持[Thymeleaf](http://www.thymeleaf.org/) 模板引擎，从服务端渲染页面

#### 服务端技术栈

- 支持spring boot 简化spring配置
- 支持maven、gradle，构建、测试、运行程序
- 支持多配置文件(默认`dev`,`prod`)
- spring security
- spring mvc REST + jackson
- spring websocket
- spring data jpa + Bean Validation
- 使用liquibase管理数据库表结构变更版本
- 支持elasticsearch，进行应用内搜素
- 支持mongoDB 、Couchbase、Cassandra等NoSQL
- 支持h2db,pgsql,mysql,meriadb,sqlserver,oracle等关系型sql
- 支持kafka mq
- 使用 zuul或者traefik作为http理由
- 使用eureka或consul进行服务发现
- 支持ehcache,hazelcast,infinispan等缓存框架
- 支持基于hazelcast的httpsession集群
- 数据源使用HikariCP连接池
- 生成Dockerfile,docker-compose.yml
- 支持云服务商,AWS,Cloud Foundry,Heroku,Kubernetes,Openshift,Docker ...
- 支持统一配置中心



其实真正用过就会发现，jhipster支持的不止列表中描述的这些。

如果不会用yarn或者不方便用命令行生成项目，可以使用[jhipster online](https://start.jhipster.tech/)

如果想学习jhipster，可以参考我在公司推广jhipster时写的一本gitbook [jhipster开发笔记](https://jh.jiankangsn.com/)

同时，值得一提的是，jhipster也支持通过 `jhipster rancher-compose` 命令来生成`rancher-compose.yml`和`docker-compose.yml` 参见 [[BETA] Deploying to Rancher](http://www.jhipster.tech/rancher/)



对于小团队落地微服务，可以考虑使用jhipster来生成项目，能够极大的提高效率。基本上可以视作jhipster是一套基于spring boot的最佳实践(不仅支持微服务，也支持单体式应用)。

对于想学习spring boot或者spring cloud的也建议了解一下jhipster，好过独自摸索



jhipster依赖的技术框架版本基本都是最新稳定版，版本更新比较及时，基本上一月一个版本，对github上的issues和pr响应比较及时(一般在24小时内)



### Service Mesh

我司是从16年八九月份开始拆分单体服务，彼时国内spring cloud，微服务等相关资料较少，国内流行dubbo(那会已经断更1年多了，虽然现在复更，但是对其前景不太看好)

从17年开始，圈内讨论spring cloud的渐渐多起来了，同时市面上也有了介绍spring cloud的书籍，比如周立的[Spring Cloud与Docker微服务架构实战](https://item.jd.com/12168358.html), 翟永超的[Spring Cloud微服务实战](https://item.jd.com/12172344.html) 等



但是用了spring cloud后，感觉spring cloud太复杂了(如果用了jhipster情况会好点)，并没有实现微服务的初衷

- 跟语言，框架无关:局限于java
- 隐藏底层细节，需要学习zuul路由，eureka注册中心，configserver配置中心，需要熔断，降级，需要实现分布式跟踪...


在这种情况下，16年，国外buoyant公司提出Service Mesh概念，基于scala创建了linkerd项目。

service mesh 的设想就是，让开发人员专注于业务，不再分心于基础设施。


目前主流框架

1. [istio](https://istio.io/) 背靠google，ibm，后台硬，前景广阔
2. [conduit ](https://conduit.io/) 跟linkerd是一个公司的，使用Rust语言开发，proxy消耗不到10M内存，p99控制在毫秒内
3. [linkerd](https://linkerd.io/) 商用企业较多，国内我知道的有豆瓣
4. [envoy](https://www.envoyproxy.io/) 国内腾讯在用


其中istio和conduit都不太成熟，而linkerd和envoy都有商用案例，较为成熟。长远来看，更看好 istio和conduit



扩展阅读资料


[官方文档|ServiceMesh服务网格Istio面板组件&设计目标](http://blog.shurenyun.com/untitled-102/)

[演讲实录 | Service Mesh 时代的选边与站队（附PPT下载）](http://www.servicemesh.cn/?/article/25)

[Service Mesh：下一代微服务](https://servicemesh.gitbooks.io/awesome-servicemesh/mesh/2017/service-mesh-next-generation-of-microservice/)



**注意**

需要根据公司、团队实际情况理性选择框架，目前service mesh还处于垦荒阶段，而spring cloud或者dubbo还没到彻底过时的程度，建议持续关注，不建议立刻上马，如果已经落地了相关的微服务技术，不要盲目跟风，在可接受学习成本和开发成本情况下，可以考虑研究一下service mesh。

如果使用的是spring框架的话，建议抛开spring cloud，直接spring boot+service mesh，更清爽一些



