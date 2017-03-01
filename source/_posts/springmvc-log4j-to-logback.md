---
title: Spring3.0 Log4j转logback
date: 2017-02-28 14:54:44
tags: [springmvc,logback,log4j,log]
categories: [编程]
---
公司项目用的还是`Spring3.0.5`,而目前`Spring5.0 M3`已发布。。。

为啥选择 `logback` 而不是`log4j2`

原因

1. log4j2 不支持动态改变logger的级别(生产环境不利于定位问题)
2. log4j2 的JSONLayout 内置字段较少，且不支持自定义。

而且log4j2引以为傲的领先logback 10倍的吞吐量的情况在最新版本中(1.2.0+)已经不存在了。具体参见(需翻墙) [FileAppender throughput
](https://docs.google.com/spreadsheets/d/1cpb5D7qnyye4W0RTlHUnXedYK98catNZytYIu5D91m0/edit#gid=0)

本文主要讲解，如何将spring3.0.5(非maven)由log4j迁移到slf4j+logback1.2.1

### Maven

`pom.xml`中关键部分代码

```xml
    <properties>
        <!-- log相关 -->
        <slf4j.version>1.7.24</slf4j.version>
        <logback.version>1.2.1</logback.version>
        
        <!-- Spring监听 -->
        <logback-ext-spring.version>0.1.4</logback-ext-spring.version>
        
        <!-- logback的logstash插件 -->
        <logstash-logback-encoder.version>4.8</logstash-logback-encoder.version>
        <!-- 可以略去jackson的依赖， logstash-logback-encoder自带的版本较低，所以手动指定jackson版本-->
        <jackson.version>2.8.6</jackson.version>
        
        <!-- 项目使用UTF-8字符集  -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <maven.compiler.encoding>UTF-8</maven.compiler.encoding>
    </properties>

    <dependencies>
        <!-- slf4j统一log接口 -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>

        <!-- slf4j接管 Apache Commons Logging -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        
        <!-- slf4j接管log4j -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>log4j-over-slf4j</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        
        <!-- logback的Spring监听 -->
        <dependency>
            <groupId>org.logback-extensions</groupId>
            <artifactId>logback-ext-spring</artifactId>
            <version>${logback-ext-spring.version}</version>
        </dependency>
        
        <!-- slf4j日志接口，logback具体实现 -->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-core</artifactId>
            <version>${logback.version}</version>
        </dependency>
        
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback.version}</version>
        </dependency>
        
        <!-- logback 日志输出到logstash的插件 -->
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>${logstash-logback-encoder.version}</version>
        </dependency>
        
        <!-- logstash-logback-encoder依赖的jackson版本较旧 -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>${jackson.version}</version>
        </dependency>

        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>
    </dependencies>
```

### 非Maven项目
(有个小技巧，本地配有maven环境的情况下，将上面的关键代码贴到`pom.xml`保存成文件,cmd进入到pom.xml同目录，运行  `mvn dependency:copy-dependencies -DoutputDirectory=lib` 会自动将依赖包，复制到当前`lib`目录下)
从[中央仓库下载](http://mvnrepository.com/)下列jar 到Spring MVC 项目的lib目录
```
jackson-annotations-2.8.0.jar
jackson-core-2.8.6.jar
jackson-databind-2.8.6.jar
jcl-over-slf4j-1.7.24.jar
log4j-over-slf4j-1.7.24.jar
logback-classic-1.2.1.jar
logback-core-1.2.1.jar
logback-ext-spring-0.1.4.jar
logstash-logback-encoder-4.8.jar
slf4j-api-1.7.24.jar
```

### 解决jar冲突

不管是maven还是非maven项目都需要删除类似`log4j.jar`,`slf4j-log4j12-xxx.jar`,旧版本的`slf4j-api-xxx.jar`和`commons-logging.jar` 确保不会有jar冲突



### 解决问题代码

删除项目自定义的一些log工具类，e.g. `StdoutListener`,`MyDailyRollingFileAppender`

### 修改web.xml

#### 删除log4j相关配置

删除以下代码

```xml
<context-param>
    <param-name>log4jConfigLocation</param-name>
    <param-value>/WEB-INF/properties/log4j.xml</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
</listener>
```
删除相关的`log4j.xml`文件

#### 添加logback相关配置
```xml
<context-param>
    <param-name>logbackConfigLocation</param-name>
    <param-value>WEB-INF/config/logback.xml</param-value>
</context-param>

<listener>
     <listener-class>ch.qos.logback.ext.spring.web.LogbackConfigListener</listener-class>
</listener>
```

### logback.xml配置

将下面的配置文件保存到 WEB-INF/config/logback.xml,注意修改项目名，logstash等相关配置

```xml
<?xml version="1.0" encoding="UTF-8"?>

<configuration scan="false" scanPeriod="60 seconds" debug="false">

    <!-- log输出目录 -->
    <property name="LOG_HOME" value="D:/logtest" />
    <!-- 项目名称 -->
    <property name="APP_NAME" value="logtest" />
    <!-- 项目端口号 -->
    <property name="APP_PORT" value="8080" />
    
    <!-- 控制台和文件的日志格式 -->
    <!-- %method和%line性能较低，如果不太介意打印的方法和行号，强烈建议取消 -->
    <property name="CONSOLE_LOG_PATTERN" value="%date{HH:mm:ss.SSS}[%-5level]%logger.%method#%line - %msg%n" />
    <property name="FILE_LOG_PATTERN" value="%date{HH:mm:ss.SSS}[%-5level]%logger.%method#%line - %msg%n" />
    
    <!-- Logstash 服务器地址和端口 -->
    <property name="LOGSTASH_SERVER" value="" />
    <property name="LOGSTASH_PORT" value="" />
    

    <logger name="org.springframework" level="WARN" />
    <logger name="org.springframework.web" level="WARN" />
    <logger name="org.springframework.security" level="WARN" />
    <logger name="org.springframework.cache" level="WARN" />
    <logger name="org.springframework.beans" level="WARN" />
    <logger name="com.shunneng.logtest" level="DEBUG" />

    <!-- 输出日志到控制台 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        
        <!-- 控制台输出性能较低。只打印ERRROR,其他信息从日志或者elasticsearch查询 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>utf8</charset>
        </encoder>
    </appender>

    <!-- 输出日志到文件  -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 文件名称 -->
        <file>${LOG_HOME}/${APP_NAME}.log</file>
        
        <!-- 编码字符集和日志格式 -->
        <encoder>
            <charset>utf-8</charset>
            <pattern>${FILE_LOG_PATTERN}</pattern>
        </encoder>
        
        <!-- 日志过大后，滚动输出日志 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${LOG_HOME}/${APP_NAME}.%i.log</fileNamePattern>
        </rollingPolicy>
        
        <!-- 限定单日志大小 -->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>100MB</MaxFileSize>
        </triggeringPolicy>
        
    </appender>
    
    <!-- 日志输出到日志搜集框架  -->
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashSocketAppender">
        <!-- logstash 服务地址  -->
        <host>${LOGSTASH_SERVER}</host>
        <!-- logstash 端口 -->
        <port>${LOGSTASH_PORT}</port>
        <!-- 自定义字段，增加项目名称和端口  -->
        <customFields>{"app_name":"${APP_NAME}","app_port":"${APP_PORT}"}</customFields>
    </appender>
    
    <!-- 异步批量(512)打印日志，在异常关闭时，有可能会有部分日志丢失 -->
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <appender-ref ref="FILE" />
    </appender>
    
    <!-- 允许动态修改日志级别 -->
    <contextListener class="ch.qos.logback.classic.jul.LevelChangePropagator">
        <resetJUL>true</resetJUL>
    </contextListener>
    
    <!-- 默认输出INFO级别日志 -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="ASYNC" />
        <appender-ref ref="LOGSTASH" />
    </root>

</configuration>

```

### Java改造

使用了`jcl-over-slf4j`和`log4j-over-slf4j`后原有方法不需要变更。但是建议在允许的情况下。改成slf4j的方法

```java
...
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
...

private final Logger logger = LoggerFactory.getLogger(Demo.class);

log.info("这是一个{}信息","info"); //输出:这是一个info信息
...
```

不建议使用logger.info("这是一个"+"info"+"信息");

原因在于，假设该logger设置的是error级别，程序走到这会先将输出信息toString后并拼接，但是因为不是error级别的，所以不输出。导致性能上的浪费。
需要改成

```java
if (logger.isInfoEnabled()){
    logger.info("这是一个"+"info"+"信息");
}
```

当然如果是常量字符串拼接，在编译时会自动优化成 `这是一个info信息`但是对于变量拼接，字符串拼接的缺点就体现出来了。（感兴趣的可以自行百度 string stringbuilder stringbuffer区别）

所以，建议使用 `{}`进行占位输出。如果是变量很多，使用`Object[]`

### 规范


**强烈建议阅读此文** [写给开发者：记录日志的10个建议](http://blog.jobbole.com/52018/) 英语原文(需翻墙)[The 10 Commandments of Logging](http://www.masterzen.fr/2013/01/13/the-10-commandments-of-logging/)

摘录其中部分内容
> ####2. 你应在适当级别上进行log

>TRACE level: 如果使用在生产环境中，这是一个代码异味(code smell)。它可以用于开发过程中追踪bug，但不要提交到你的版本控制系统

>DEBUG level: 把一切东西都记录在这里。这在debug过程中最常用到。我主张在进入生产阶段前减少debug语句的数量，只留下最有意义的部分，在调试(troubleshooting)的时候激活。

>INFO level: 把用户行为(user-driven)和系统的特定行为(例如计划任务…)

>NOTICE level: 这是生产环境中使用的级别。把一切不认为是错误的，可以记录的事件都log起来

>WARN level: 记录在这个级别的事件都有可能成为一个error。例如，一次调用数据库使用的时间超过了预设时间，或者内存缓存即将到达容量上限。这可以让你适当地发出警报，或者在调试时更好地理解系统在failure之前做了些什么

>ERROR level: 把每一个错误条件都记录在这。例如API调用返回了错误，或是内部错误条件

>FATAL level: 末日来了。它极少被用到，在实际程序中也不应该出现多少。在这个级别上进行log意味着程序要结束了。例如一个网络守护进程无法bind到socket上，那么它唯一能做的就只有log到这里，然后退出运行。

> ####4. 你应该写有意义的log

> ####6. 你应该给log带上上下文

> ####7. 你应该用机器可解析的格式来打日志

对于需要打印的对象，一定注意重载对象的toString方法，或者使用commons-lang3包下的 `ReflectionToStringBuilder.toString()`和`new ToStringBuilder()`

其中  `ReflectionToStringBuilder.toString()` 打印的类似 `lang.Foo@c2a132[name=foo,age=88,bar=lang.Bar@e102dc[name=bar]] `

而 `new ToStringBuilder()`可以只打印部分属性
```java
new ToStringBuilder(this, ToStringStyle.MULTI_LINE_STYLE)
    .append("name", name)
    .append("age", age)
    .append("bar", bar)
    .toString()
```