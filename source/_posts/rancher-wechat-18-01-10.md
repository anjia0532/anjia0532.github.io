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



## 使用jhipster创建微服务

### [安装node,yarn](https://jh.jiankangsn.com/install.html)

**注意**

如果是windows nodejs 需要安装v7.x，因为注册中心和网关需要用到node-sass@4.5.0，但是github上的node-sass的rebuild只有v7.x(process 51) 版本的，而自己构建太反人类了。如果是linux，可以尝试高版本的

手动下载 `https://github.com/Medium/phantomjs/releases/download/v2.1.1/phantomjs-2.1.1-windows.zip`

解压并添加到环境变量里

## 安装jhipster



### 下载注册中心

下载并运行 [注册中心-jhipster-registry](https://github.com/jhipster/jhipster-registry) 详细文档，参见 [The JHipster Registry](http://www.jhipster.tech/jhipster-registry/)

```bash
$ git clone https://github.com/jhipster/jhipster-registry.git
$ cd jhipster-registry
$ yarn install
$ mvnw
##....
----------------------------------------------------------
        Application 'jhipster-registry' is running! Access URLs:
        Local:          http://localhost:8761
        External:       http://xx.xx.xx.xx:8761
        Profile(s):     [swagger, dev, native]
----------------------------------------------------------
```

浏览器访问 `http://localhost:8761`  初始用户名密码均为 `admin`

![](http://ww1.sinaimg.cn/large/afaffa71ly1fn813ofaf2j21fe0q4wgt.jpg)



spring config server,统一配置中心，可以统一管理不同环境的数据库地址，用户名，密码等敏感数据

![](http://ww1.sinaimg.cn/large/afaffa71ly1fn813ofbxjj21fp0ofta7.jpg)



### 创建网关

创建api网关，参见 [Creating an application](http://www.jhipster.tech/creating-an-app/) 和[The JHipster API Gateway](http://www.jhipster.tech/api-gateway/)

```bash
$ yarn global add generator-jhipster
$ mkdir gateway
$ cd gateway
$ yo jhipster


        ██╗ ██╗   ██╗ ████████╗ ███████╗   ██████╗ ████████╗ ████████╗ ███████╗
        ██║ ██║   ██║ ╚══██╔══╝ ██╔═══██╗ ██╔════╝ ╚══██╔══╝ ██╔═════╝ ██╔═══██╗
        ██║ ████████║    ██║    ███████╔╝ ╚█████╗     ██║    ██████╗   ███████╔╝
  ██╗   ██║ ██╔═══██║    ██║    ██╔════╝   ╚═══██╗    ██║    ██╔═══╝   ██╔══██║
  ╚██████╔╝ ██║   ██║ ████████╗ ██║       ██████╔╝    ██║    ████████╗ ██║  ╚██╗
   ╚═════╝  ╚═╝   ╚═╝ ╚═══════╝ ╚═╝       ╚═════╝     ╚═╝    ╚═══════╝ ╚═╝   ╚═╝

                            http://www.jhipster.tech

Welcome to the JHipster Generator v4.13.2
 _______________________________________________________________________________________________________________

  If you find JHipster useful consider supporting our collective https://opencollective.com/generator-jhipster
  Documentation for creating an application: http://www.jhipster.tech/creating-an-app/
 _______________________________________________________________________________________________________________

Application files will be generated in folder: G:\jh\gateway
? Which *type* of application would you like to create? Microservice gateway
? What is the base name of your application? gateway
? As you are running in a microservice architecture, on which port would like your server to run? It should be unique to avoid port conflicts. 8080
? What is your default Java package name? com.anjia.gateway
? Which service discovery server do you want to use? JHipster Registry (uses Eureka, provides Spring Cloud Config support and monitoring dashboards)
? Which *type* of authentication would you like to use? JWT authentication (stateless, with a token)
? Which *type* of database would you like to use? SQL (H2, MySQL, MariaDB, PostgreSQL, Oracle, MSSQL)
? Which *production* database would you like to use? MySQL
? Which *development* database would you like to use? H2 with disk-based persistence
? Do you want to use Hibernate 2nd level cache? Yes
? Would you like to use Maven or Gradle for building the backend? Maven
? Which other technologies would you like to use?
? Which *Framework* would you like to use for the client? Angular 5
? Would you like to enable *SASS* support using the LibSass stylesheet preprocessor? No
? Would you like to enable internationalization support? Yes
? Please choose the native language of the application English
? Please choose additional languages to install
? Besides JUnit and Karma, which testing frameworks would you like to use?
? Would you like to install other generators from the JHipster Marketplace? No
$ mvnw

----------------------------------------------------------
        Application 'gateway' is running! Access URLs:
        Local:          http://localhost:8080
        External:       http://xx.xx.xx.xx:8080
        Profile(s):     [swagger, dev]
----------------------------------------------------------
```



### 创建服务

创建服务 参考 [Creating an application](http://www.jhipster.tech/creating-an-app/)

```bash
$ mkdir foo
$ cd foo
$ yo jhipster


        ██╗ ██╗   ██╗ ████████╗ ███████╗   ██████╗ ████████╗ ████████╗ ███████╗
        ██║ ██║   ██║ ╚══██╔══╝ ██╔═══██╗ ██╔════╝ ╚══██╔══╝ ██╔═════╝ ██╔═══██╗
        ██║ ████████║    ██║    ███████╔╝ ╚█████╗     ██║    ██████╗   ███████╔╝
  ██╗   ██║ ██╔═══██║    ██║    ██╔════╝   ╚═══██╗    ██║    ██╔═══╝   ██╔══██║
  ╚██████╔╝ ██║   ██║ ████████╗ ██║       ██████╔╝    ██║    ████████╗ ██║  ╚██╗
   ╚═════╝  ╚═╝   ╚═╝ ╚═══════╝ ╚═╝       ╚═════╝     ╚═╝    ╚═══════╝ ╚═╝   ╚═╝

                            http://www.jhipster.tech

Welcome to the JHipster Generator v4.13.2
 _______________________________________________________________________________________________________________

  If you find JHipster useful consider supporting our collective https://opencollective.com/generator-jhipster
  Documentation for creating an application: http://www.jhipster.tech/creating-an-app/
 _______________________________________________________________________________________________________________

Application files will be generated in folder: G:\jh\foo
? Which *type* of application would you like to create? Microservice application
? What is the base name of your application? foo
? As you are running in a microservice architecture, on which port would like your server to run? It should be unique to avoid port conflicts. 8081
? What is your default Java package name? com.anjia.foo
? Which service discovery server do you want to use? JHipster Registry (uses Eureka, provides Spring Cloud Config support and monitoring dashboards)
? Which *type* of authentication would you like to use? JWT authentication (stateless, with a token)
? Which *type* of database would you like to use? SQL (H2, MySQL, MariaDB, PostgreSQL, Oracle, MSSQL)
? Which *production* database would you like to use? MySQL
? Which *development* database would you like to use? H2 with disk-based persistence
? Do you want to use the Spring cache abstraction? Yes, with the Hazelcast implementation (distributed cache, for multiple nodes)
? Do you want to use Hibernate 2nd level cache? Yes
? Would you like to use Maven or Gradle for building the backend? Maven
? Which other technologies would you like to use?
? Would you like to enable internationalization support? Yes
? Please choose the native language of the application Chinese (Simplified)
? Please choose additional languages to install
? Besides JUnit and Karma, which testing frameworks would you like to use?
? Would you like to install other generators from the JHipster Marketplace? No

$ mvnw
----------------------------------------------------------
        Application 'foo' is running! Access URLs:
        Local:          http://localhost:8081
        External:       http://xx.xx.xx.xx:8081
        Profile(s):     [swagger, dev]
----------------------------------------------------------
```



访问 `http://localhost:8080/#/docs` 默认用户名密码均为 `admin` ，使用`swagger`管理api文档，开发时，仅需要添加对应的注解，即可自动生成文档，解决了传统通过word，pdf等管理接口时，文档更新不及时等问题。并且可以通过`try it`可以直接调用接口，避免了接口调试时使用`curl`,`postman`等工具

![](http://ww1.sinaimg.cn/large/afaffa71ly1fn813ofjgkj20y40l7dgs.jpg)


至此，已经创建了一个简单微服务（jhipster-registry是注册中心，gateway是网关，foo是具体的功能模块）。

### 创建实体

参考文档 [Creating an entity](http://www.jhipster.tech/creating-an-entity/)



jhipster支持通过命令行创建实体，也支持uml或jdl生成实体，为了省事，此处使用 官方`jdl-studio` 的默认jdl文件 https://start.jhipster.tech/jdl-studio/

![](http://ww1.sinaimg.cn/large/afaffa71ly1fn81m4vh86j20dy02kdfu.jpg)



```bash
$ yo jhipster:import-jdl /path/to/jdl-studio/jhipster-jdl.jh
The jdl is being parsed.
Writing entity JSON files.
Updated entities are: Region,Country,Location,Department,Task,Employee,Job,JobHistory
Generating entities.

Found the .jhipster/Region.json configuration file, entity can be automatically generated!


The entity Region is being updated.


Found the .jhipster/Country.json configuration file, entity can be automatically generated!


The entity Country is being updated.


Found the .jhipster/Location.json configuration file, entity can be automatically generated!


The entity Location is being updated.


Found the .jhipster/Department.json configuration file, entity can be automatically generated!


The entity Department is being updated.


Found the .jhipster/Task.json configuration file, entity can be automatically generated!


The entity Task is being updated.


Found the .jhipster/Employee.json configuration file, entity can be automatically generated!


The entity Employee is being updated.


Found the .jhipster/Job.json configuration file, entity can be automatically generated!


The entity Job is being updated.


Found the .jhipster/JobHistory.json configuration file, entity can be automatically generated!


The entity JobHistory is being updated.

   create src\main\resources\config\liquibase\changelog\20180107064934_added_entity_Region.xml
   create src\main\java\com\anjia\foo\domain\Region.java
   create src\main\java\com\anjia\foo\repository\RegionRepository.java
   create src\main\java\com\anjia\foo\web\rest\RegionResource.java
   create src\main\java\com\anjia\foo\service\RegionService.java
   create src\main\java\com\anjia\foo\service\impl\RegionServiceImpl.java
   create src\main\java\com\anjia\foo\service\dto\RegionDTO.java
   create src\main\java\com\anjia\foo\service\mapper\EntityMapper.java
   create src\main\java\com\anjia\foo\service\mapper\RegionMapper.java
   create src\test\java\com\anjia\foo\web\rest\RegionResourceIntTest.java
 conflict src\main\resources\config\liquibase\master.xml
? Overwrite src\main\resources\config\liquibase\master.xml? overwrite
    force src\main\resources\config\liquibase\master.xml
   create src\main\resources\config\liquibase\changelog\20180107064935_added_entity_Country.xml
   create src\main\resources\config\liquibase\changelog\20180107064935_added_entity_constraints_Country.xml
   create src\main\java\com\anjia\foo\domain\Country.java
   create src\main\java\com\anjia\foo\repository\CountryRepository.java
   create src\main\java\com\anjia\foo\web\rest\CountryResource.java
   create src\main\java\com\anjia\foo\service\CountryService.java
   create src\main\java\com\anjia\foo\service\impl\CountryServiceImpl.java
   create src\main\java\com\anjia\foo\service\dto\CountryDTO.java
   create src\main\java\com\anjia\foo\service\mapper\CountryMapper.java
   create src\test\java\com\anjia\foo\web\rest\CountryResourceIntTest.java
   create src\main\resources\config\liquibase\changelog\20180107064936_added_entity_Location.xml
   create src\main\resources\config\liquibase\changelog\20180107064936_added_entity_constraints_Location.xml
   create src\main\java\com\anjia\foo\domain\Location.java
   create src\main\java\com\anjia\foo\repository\LocationRepository.java
   create src\main\java\com\anjia\foo\web\rest\LocationResource.java
   create src\main\java\com\anjia\foo\service\LocationService.java
   create src\main\java\com\anjia\foo\service\impl\LocationServiceImpl.java
   create src\main\java\com\anjia\foo\service\dto\LocationDTO.java
   create src\main\java\com\anjia\foo\service\mapper\LocationMapper.java
   create src\test\java\com\anjia\foo\web\rest\LocationResourceIntTest.java
   create src\main\resources\config\liquibase\changelog\20180107064937_added_entity_Department.xml
   create src\main\resources\config\liquibase\changelog\20180107064937_added_entity_constraints_Department.xml
   create src\main\java\com\anjia\foo\domain\Department.java
   create src\main\java\com\anjia\foo\repository\DepartmentRepository.java
   create src\main\java\com\anjia\foo\web\rest\DepartmentResource.java
   create src\main\java\com\anjia\foo\service\DepartmentService.java
   create src\main\java\com\anjia\foo\service\impl\DepartmentServiceImpl.java
   create src\main\java\com\anjia\foo\service\dto\DepartmentDTO.java
   create src\main\java\com\anjia\foo\service\mapper\DepartmentMapper.java
   create src\test\java\com\anjia\foo\web\rest\DepartmentResourceIntTest.java
   create src\main\resources\config\liquibase\changelog\20180107064938_added_entity_Task.xml
   create src\main\java\com\anjia\foo\domain\Task.java
   create src\main\java\com\anjia\foo\repository\TaskRepository.java
   create src\main\java\com\anjia\foo\web\rest\TaskResource.java
   create src\main\java\com\anjia\foo\service\TaskService.java
   create src\main\java\com\anjia\foo\service\impl\TaskServiceImpl.java
   create src\main\java\com\anjia\foo\service\dto\TaskDTO.java
   create src\main\java\com\anjia\foo\service\mapper\TaskMapper.java
   create src\test\java\com\anjia\foo\web\rest\TaskResourceIntTest.java
   create src\main\resources\config\liquibase\changelog\20180107064939_added_entity_Employee.xml
   create src\main\resources\config\liquibase\changelog\20180107064939_added_entity_constraints_Employee.xml
   create src\main\java\com\anjia\foo\domain\Employee.java
   create src\main\java\com\anjia\foo\repository\EmployeeRepository.java
   create src\main\java\com\anjia\foo\web\rest\EmployeeResource.java
   create src\main\java\com\anjia\foo\service\dto\EmployeeDTO.java
   create src\main\java\com\anjia\foo\service\mapper\EmployeeMapper.java
   create src\test\java\com\anjia\foo\web\rest\EmployeeResourceIntTest.java
   create src\main\resources\config\liquibase\changelog\20180107064940_added_entity_Job.xml
   create src\main\resources\config\liquibase\changelog\20180107064940_added_entity_constraints_Job.xml
   create src\main\java\com\anjia\foo\domain\Job.java
   create src\main\java\com\anjia\foo\repository\JobRepository.java
   create src\main\java\com\anjia\foo\web\rest\JobResource.java
   create src\main\java\com\anjia\foo\service\dto\JobDTO.java
   create src\main\java\com\anjia\foo\service\mapper\JobMapper.java
   create src\test\java\com\anjia\foo\web\rest\JobResourceIntTest.java
   create src\main\resources\config\liquibase\changelog\20180107064941_added_entity_JobHistory.xml
   create src\main\resources\config\liquibase\changelog\20180107064941_added_entity_constraints_JobHistory.xml
   create src\main\java\com\anjia\foo\domain\JobHistory.java
   create src\main\java\com\anjia\foo\repository\JobHistoryRepository.java
   create src\main\java\com\anjia\foo\web\rest\JobHistoryResource.java
   create src\main\java\com\anjia\foo\service\JobHistoryService.java
   create src\main\java\com\anjia\foo\service\impl\JobHistoryServiceImpl.java
   create src\main\java\com\anjia\foo\service\dto\JobHistoryDTO.java
   create src\main\java\com\anjia\foo\service\mapper\JobHistoryMapper.java
   create src\test\java\com\anjia\foo\web\rest\JobHistoryResourceIntTest.java
   create src\main\java\com\anjia\foo\domain\enumeration\Language.java
```

重启 `foo`服务,再次访问 `http://localhost:8080/#/docs` 发现多了很多接口

![](http://ww1.sinaimg.cn/large/afaffa71ly1fn81x40lzzj21240oo412.jpg)



通过swagger ui，找到 `region-resource` 找到 `POST /api/regions ` 创建一个名为`test`的`regison`

![](http://ww1.sinaimg.cn/large/afaffa71ly1fn8249j1itj20rj0mtaax.jpg)

点 `try it out!` 然后浏览器打开 `h2 数据库` `http://localhost:8081/h2-console` 

![](http://ww1.sinaimg.cn/large/afaffa71ly1fn825ejmpxj20g90b3t8x.jpg)
![](http://ww1.sinaimg.cn/large/afaffa71ly1fn825ehnhnj20op0bp755.jpg)

查询 `REGION`表，数据已经插入成功.



## 将服务发布到rancher

参见文档 [[BETA] Deploying to Rancher](http://www.jhipster.tech/rancher/) ,jhipster支持发布到 [Cloud Foundry](http://www.jhipster.tech/cloudfoundry/) ,[Heroku](http://www.jhipster.tech/heroku/),[Kubernetes](http://www.jhipster.tech/kubernetes/),[Openshift](http://www.jhipster.tech/openshift/),[Rancher](http://www.jhipster.tech/rancher/),[AWS](http://www.jhipster.tech/aws/),[Boxfuse](http://www.jhipster.tech/boxfuse/)

```bash
 $ mkdir docker
 $ cd docker
 $ yo jhipster rancher-rancher


        ██╗ ██╗   ██╗ ████████╗ ███████╗   ██████╗ ████████╗ ████████╗ ███████╗
        ██║ ██║   ██║ ╚══██╔══╝ ██╔═══██╗ ██╔════╝ ╚══██╔══╝ ██╔═════╝ ██╔═══██╗
        ██║ ████████║    ██║    ███████╔╝ ╚█████╗     ██║    ██████╗   ███████╔╝
  ██╗   ██║ ██╔═══██║    ██║    ██╔════╝   ╚═══██╗    ██║    ██╔═══╝   ██╔══██║
  ╚██████╔╝ ██║   ██║ ████████╗ ██║       ██████╔╝    ██║    ████████╗ ██║  ╚██╗
   ╚═════╝  ╚═╝   ╚═╝ ╚═══════╝ ╚═╝       ╚═════╝     ╚═╝    ╚═══════╝ ╚═╝   ╚═╝

                            http://www.jhipster.tech

Welcome to the JHipster Generator v4.13.2
 _______________________________________________________________________________________________________________

  If you find JHipster useful consider supporting our collective https://opencollective.com/generator-jhipster
  Documentation for creating an application: http://www.jhipster.tech/creating-an-app/
 _______________________________________________________________________________________________________________

Application files will be generated in folder: G:\jh\docker
? What is the base name of your application? (docker)
G:\jh\docker
λ  jhipster rancher-compose
Using JHipster version installed globally
Executing jhipster:rancher-compose
Options:
🐮 [BETA] Welcome to the JHipster Rancher Compose Generator 🐮
Files will be generated in folder: G:\jh\docker
? Which *type* of application would you like to deploy? Microservice application
? Enter the root directory where your gateway(s) and microservices are located ../
2 applications found at G:\jh\

? Which applications do you want to include in your configuration? foo, gateway
? Do you want to setup monitoring for your applications ? Yes, for logs and metrics with the JHipster Console (based on ELK and Zipkin)
JHipster registry detected as the service discovery and configuration provider used by your apps
? Enter the admin password used to secure the JHipster Registry admin
? Would you like to enable rancher load balancing support? Yes
? What should we use for the base Docker repository name?
? What command should we use for push Docker image to repository? docker push

Checking Docker images in applications' directories...
ls: no such file or directory: G:/jh/foo/target/docker
ls: no such file or directory: G:/jh/gateway/target/docker
   create rancher-compose.yml
   create docker-compose.yml
   create registry-config-sidekick\Dockerfile
   create registry-config-sidekick\application.yml


Rancher Compose configuration generated with missing images!
To generate the missing Docker image(s), please run:
  ./mvnw verify -Pprod dockerfile:build in G:\jh\foo
  ./mvnw verify -Pprod dockerfile:build in G:\jh\gateway

WARNING! You will need to push your image to a registry. If you have not done so, use the following commands to tag and push the images:
docker push foo
docker push gateway
Congratulations, JHipster execution is complete!
```



#### rancher-compose.yml

```yaml
version: '2'
services:
    lb:
        # load balancer container
        scale: 1
        load_balancer_config:
          name: lb config
        health_check:
          port: 42
          interval: 2000
          unhealthy_threshold: 3
          healthy_threshold: 2
          response_timeout: 2000
    foo-app:
        scale: 1
    foo-mysql:
        scale: 1
    
    gateway-app:
        scale: 1
    gateway-mysql:
        scale: 1
    

    jhipster-registry:
        scale: 1


    jhipster-elasticsearch:
        scale: 1
    jhipster-logstash:
        scale: 1
    jhipster-console:
        scale: 1
    # Uncomment this section to enable Zipkin
    #jhipster-zipkin:
    #    scale: 1

```



#### docker-compose.yml

```yaml
version: '2'
services:
    lb:
      image: rancher/load-balancer-service
      ports:
        # Listen on public port 80 and direct traffic to private port 8080 of the service
        - 80:8080
      links:
        # Target services in the same stack will be listed as a link
        - gateway-app:gateway-app
    foo-app:
        image: foo
        environment:
            - SPRING_PROFILES_ACTIVE=prod,swagger
            - EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://admin:$${jhipster.registry.password}@jhipster-registry:8761/eureka
            - SPRING_CLOUD_CONFIG_URI=http://admin:$${jhipster.registry.password}@jhipster-registry:8761/config
            - SPRING_DATASOURCE_URL=jdbc:mysql://foo-mysql:3306/foo?useUnicode=true&characterEncoding=utf8&useSSL=false
            - JHIPSTER_SLEEP=30
            - JHIPSTER_LOGGING_LOGSTASH_ENABLED=true
            - JHIPSTER_LOGGING_LOGSTASH_HOST=jhipster-logstash
            - JHIPSTER_METRICS_LOGS_ENABLED=true
            - JHIPSTER_METRICS_LOGS_REPORT_FREQUENCY=60
            - JHIPSTER_REGISTRY_PASSWORD=admin
    foo-mysql:
        image: mysql:5.7.20
        environment:
            - MYSQL_USER=root
            - MYSQL_ALLOW_EMPTY_PASSWORD=yes
            - MYSQL_DATABASE=foo
        command: >-
            mysqld --lower_case_table_names=1 --skip-ssl --character_set_server=utf8
            --explicit_defaults_for_timestamp
    
    gateway-app:
        image: gateway
        environment:
            - SPRING_PROFILES_ACTIVE=prod,swagger
            - EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://admin:$${jhipster.registry.password}@jhipster-registry:8761/eureka
            - SPRING_CLOUD_CONFIG_URI=http://admin:$${jhipster.registry.password}@jhipster-registry:8761/config
            - SPRING_DATASOURCE_URL=jdbc:mysql://gateway-mysql:3306/gateway?useUnicode=true&characterEncoding=utf8&useSSL=false
            - JHIPSTER_SLEEP=30
            - JHIPSTER_LOGGING_LOGSTASH_ENABLED=true
            - JHIPSTER_LOGGING_LOGSTASH_HOST=jhipster-logstash
            - JHIPSTER_METRICS_LOGS_ENABLED=true
            - JHIPSTER_METRICS_LOGS_REPORT_FREQUENCY=60
            - JHIPSTER_REGISTRY_PASSWORD=admin
        ports:
            - 8080:8080
    gateway-mysql:
        image: mysql:5.7.20
        environment:
            - MYSQL_USER=root
            - MYSQL_ALLOW_EMPTY_PASSWORD=yes
            - MYSQL_DATABASE=gateway
        command: >-
            mysqld --lower_case_table_names=1 --skip-ssl --character_set_server=utf8
            --explicit_defaults_for_timestamp
    
    jhipster-registry:
        image: jhipster/jhipster-registry:v3.2.4
        #volumes:
        #    - ./central-server-config:/central-config
        # By default the JHipster Registry runs with the "dev" and "native"
        # Spring profiles.
        # "native" profile means the filesystem is used to store data, see
        # http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html
        environment:
            - SPRING_PROFILES_ACTIVE=dev,native
            - SECURITY_USER_PASSWORD=admin
            - JHIPSTER_LOGGING_LOGSTASH_ENABLED=true
            - JHIPSTER_LOGGING_LOGSTASH_HOST=jhipster-logstash
            - JHIPSTER_METRICS_LOGS_ENABLED=true
            - JHIPSTER_METRICS_LOGS_REPORTFREQUENCY=60
            - SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS=file:./config/
            # Uncomment to use a Git configuration source instead of the local filesystem
            # mounted from the registry-config-sidekick volume
            # - GIT_URI=https://github.com/jhipster/jhipster-registry/
            # - GIT_SEARCH_PATHS=central-config
        ports:
            - 8761:8761
        volumes:
          - /config
        volumes_from:
          - registry-config-sidekick
        labels:
          io.rancher.sidekicks: registry-config-sidekick
    registry-config-sidekick:
        # this docker image must be built with:
        # docker build -t registry-config-sidekick registry-config-sidekick
        image: registry-config-sidekick
        tty: true
        stdin_open: true
        command:
          - cat
        volumes:
          - config:/config
    jhipster-elasticsearch:
        image: jhipster/jhipster-elasticsearch:v2.2.1
        ports:
            - 9200:9200
            - 9300:9300
        # Uncomment this section to have elasticsearch data persisted to a volume
        #volumes:
        #   - ./log-data:/usr/share/elasticsearch/data
    jhipster-logstash:
        image: jhipster/jhipster-logstash:v2.2.1
        command: logstash -f /conf/logstash.conf
        ports:
            - 5000:5000/udp
        # Uncomment this section to have logstash config loaded from a volume
        #volumes:
        #    - ./log-conf/:/conf
    jhipster-console:
        image: jhipster/jhipster-console:v2.2.1
        ports:
            - 5601:5601
    # Uncomment this section to enable Zipkin
    #jhipster-zipkin:
    #    image: jhipster/jhipster-zipkin:v2.2.1
    #    ports:
    #        - 9411:9411
    #    environment:
    #        - ES_HOSTS=http://jhipster-elasticsearch:9200
    #        - ZIPKIN_UI_LOGS_URL=http://localhost:5601/app/kibana#/dashboard/logs-dashboard?_g=(refreshInterval:(display:Off,pause:!f,value:0),time:(from:now-1h,mode:quick,to:now))&_a=(filters:!(),options:(darkTheme:!f),panels:!((col:1,id:logs-levels,panelIndex:2,row:1,size_x:6,size_y:3,type:visualization),(col:7,columns:!(stack_trace),id:Stacktraces,panelIndex:7,row:1,size_x:4,size_y:3,sort:!('@timestamp',desc),type:search),(col:11,id:Log-forwarding-instructions,panelIndex:8,row:1,size_x:2,size_y:3,type:visualization),(col:1,columns:!(app_name,level,message),id:All-logs,panelIndex:9,row:4,size_x:12,size_y:7,sort:!('@timestamp',asc),type:search)),query:(query_string:(analyze_wildcard:!t,query:'{traceId}')),title:logs-dashboard,uiState:())

```



## Service Mesh--下一代微服务

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



