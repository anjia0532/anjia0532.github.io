
---

title: 加速和简化构建Docker(基于Google jib)

date: 2019-02-08 21:26:00 +0800

tags: [k8s,容器,docker,微服务,容器,google-jib,jib,arthas,jvm]

categories: k8s

---

<a name="61a3ec66"></a>
## 介绍

其实jib刚发布时就有关注，但是一直没有用于生产，原因有二

- 基于 [spotify/docker-maven-plugin](https://github.com/spotify/docker-maven-plugin) (原作者已经停止维护docker-maven-plugin，建议使用 [spotify/dockerfile-maven](https://github.com/spotify/dockerfile-maven))的原有流程跑的好好的，没动力换成jib
- Google jib 一直没有发布1.x ，担心其不稳定

先简单介绍一下: [google jib](https://github.com/GoogleContainerTools/jib/) 是Google于18年7月发布的一个针对Java应用的构建镜像的工具(支持[Maven](https://github.com/GoogleContainerTools/jib/blob/master/jib-maven-plugin)和[Gradle](https://github.com/GoogleContainerTools/jib/blob/master/jib-gradle-plugin)) ，好处是能够复用构建缓存，能够加快构建，减小传输体积（后文会详细讲解），并且让Java工程师不需要理解Docker相关知识就可以简单构建镜像并且发布到指定registry里（不需要docker build , tag, push）

本文会依次讲解三种java构建镜像的方法，分别是 **正统的Dockerfile** ，**spotify/dockerfile-maven** ，**Google jib** <br />
附赠 [alibaba/arthas](https://github.com/alibaba/arthas) 的集成和使用方法

<!-- more -->
<a name="424a2ad8"></a>
## 准备

- Maven3.5
- Git
- Jdk 1.8
- Docker

```bash
$ git clone https://github.com/anjia0532/jib-demo.git
$ cd jib-demo
$ mvn clean package -DskipTests
$ mkdir docker
$ cp ./target/*.jar ./docker
```

<a name="Dockerfile"></a>
## Dockerfile

创建 `./docker/Dockerfile` ,内容如下，需要注意此处为了方便理解，没有进行改进（比如限制用户，安装必要软件等）

```dockerfile
FROM openjdk:8-jdk-alpine

ADD *.jar /app.jar

EXPOSE 8080

CMD java ${JAVA_OPTS} /app.jar
```

详情参见 官方文档 [Dockerfile reference](https://docs.docker.com/engine/reference/builder/#usage)

```bash
$ cd ./docker
$ sudo docker build . -t jib-demo
$ docker images --查看本地images
$ docker tag jib-demo anjia0532/jib-demo --不写registry，则默认为docker hub registry,可以在build时，直接写
$ docker push anjia0532/jib-demo -- 推送到registry
```

<a name="5db9fd7c"></a>
### 小结

**优点：** 不需要改造pom，灵活，配合CI工具，可以不侵入项目，运维可以针对性的进行安全加固，并且可以做到标准化<br />
**缺点：** 命令复杂，Java程序员需要学习Dockerfile命令，或者运维和java沟通不畅时，时区，软件，甚至目录等都可能有出问题

<a name="50df2cce"></a>
## spotify/dockerfile-maven

需要注意，spotify/dockerfile-maven 是需要pom+Dockerfile一块用的，而docker-maven-plugin是可选的

在项目根目录创建Dockerfile，如下所示

```dockerfile
FROM openjdk:8-jdk-alpine

EXPOSE 8080

ARG JAR_FILE
ADD target/${JAR_FILE} /usr/share/myservice/myservice.jar

ENTRYPOINT ["/usr/bin/java", "-jar", "/usr/share/myservice/myservice.jar"]
```

在pom里增加dockerfile-maven-plugin到build标签里

```xml
<build>
  <plugins>
    ...
    <plugin>
      <groupId>com.spotify</groupId>
      <artifactId>dockerfile-maven-plugin</artifactId>
      <version>${dockerfile-maven-version}</version>
      <executions>
        <execution>
          <id>default</id>
          <goals>
            <goal>build</goal>
            <goal>push</goal>
          </goals>
        </execution>
      </executions>
      <configuration>
        <repository>anjia0532/dockerfile-maven-demo</repository>
        <tag>${project.version}</tag>
        <buildArgs>
          <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
        </buildArgs>
      </configuration>
    </plugin>
    ...
  <plugins>    
<build>
```

运行如下命令进行构建

```bash
$ mvn package -DskipTests
```

详情参见 官方文档 [spotify/dockerfile-maven](https://github.com/spotify/dockerfile-maven)


<a name="5db9fd7c"></a>
### 小结

**优点：** 减少了docker build & tag & push 的操作，Java程序员能够控制镜像名<br />
**缺点： **其实就是省了docker build & tag & push的操作，别的缺点一点没落不说，还得改动pom，还得要求写Dockerfile，tag只支持一个等等

<a name="ce62745d"></a>
## Google jib

修改默认settings.xml，增加registry认证,参见[Authentication Methods](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#authentication-methods)

```xml
<settings>
  ...
  <servers>
    ...
    <server>
      <id>docker_hub</id>
      <username>MY_USERNAME</username>
      <password>{MY_SECRET}</password>
    </server>
  </servers>
</settings>
```

改动pom.xml 增加 jib插件

```xml
<project>
  ...
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.google.cloud.tools</groupId>
        <artifactId>jib-maven-plugin</artifactId>
        <version>1.0.0</version>
        <configuration>
          <from>
            <!-- 如果不需要arthas可以改为 registry.hub.docker.com/openjdk:8-jdk-alpine -->
            <image>registry.hub.docker.com/hengyunabc/arthas:latest</image>
            <credHelper>docker_hub</credHelper>
          </from>
          <to>
            <image>${project.artifactId}</image>
            <tags>
              <tag>latest</tag>
              <tag>${project.version}</tag>
            </tags>
          </to>
          <container>
            <mainClass>${start-class}</mainClass>
            <ports>
              <port>8080</port>
              <port>5701/udp</port>
              <port>8563</port>
            </ports>
            <entrypoint>
              <shell>sh</shell>
              <option>-c</option>
              <arg>java -cp /app/resources/:/app/classes/:/app/libs/* com.example.demo.DemoApplication</arg>
            </entrypoint>
            <appRoot>/app</appRoot>
            <useCurrentTimestamp>true</useCurrentTimestamp>
          </container>
        </configuration>
      </plugin>
      ...
    </plugins>
  </build>
  ...
</project>
```

执行`mvn clean compile jib:dockerBuild` 构建docker镜像

**注意：** <br />
pom中用的是 `registry.hub.docker.com/hengyunabc/arthas:latest` 是 [alibaba/arthas](https://github.com/alibaba/arthas/blob/master/README_CN.md) (阿里开源的一个Java诊断工具，便于线上调试)封装的docker镜像，如果不需要可以改成 `registry.hub.docker.com/openjdk:8-jdk-alpine`

```bash
$ docker run -d --init -p8563:8563 --name demo demo:latest
## 下面是启动arthas，如果使用的是openjdk镜像请勿执行
$ docker exec -it demo /bin/sh
$ jid=$(jps | grep App | awk '{print $1}')
$ java -jar /opt/arthas/arthas-boot.jar --target-ip 0.0.0.0 ${jid}
```

如果使用了arthas镜像，可以访问 [http://ip:8563](http://ip:8563) ，在页面上填上宿主ip，点击Connect， 然后参考[Arthas/命令列表](https://alibaba.github.io/arthas/commands.html) 了解Arthas命令用法

![snipaste20190208_201332.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1549628164482-56f9cb66-b8bc-42db-82a3-ceb8717460bd.png#align=left&display=inline&height=340&linkTarget=_blank&name=snipaste20190208_201332.png&originHeight=612&originWidth=1343&size=85193&width=746#align=left&display=inline&height=340&originHeight=612&originWidth=1343&width=746)

<a name="5db9fd7c"></a>
### 小结

**优点：** 充分利用缓存，加快构建，不强制依赖docker daemon，依赖简单（在maven或gradle增加插件即可）<br />
**缺点：** 不支持Docker RUN 命令(jib官方建议将run封装到base镜像)，对entrypoint和cmd支持不太好(alpine默认不支持多任务，跑java应用默认是pid 1 ，运行jmap等命令会报错，参考 [jmap not happy on alpine](https://github.com/docker-library/openjdk/issues/76) )

<a name="d3f22cc0"></a>
### jib 缓存策略

项目每次发布实际上变更的代码量不大，尤其依赖的jar变动的可能性较小，如果按照前两种方案构建镜像，会导致每次都全量构建，会导致存储和带宽资源浪费。

> <a name="396d6264"></a>
## **Jib 如何让开发变得更美好**
> Jib 利用了 Docker 镜像的分层机制，将其与构建系统集成，并通过以下方式优化 Java 容器镜像的构建：
> 1. 简单——Jib 使用 Java 开发，并作为 Maven 或 Gradle 的一部分运行。你不需要编写 Dockerfile 或运行 Docker 守护进程，甚至无需创建包含所有依赖的大 JAR 包。因为 Jib 与 Java 构建过程紧密集成，所以它可以访问到打包应用程序所需的所有信息。在后续的容器构建期间，它将自动选择 Java 构建过的任何变体。
> 1. 快速——Jib 利用镜像分层和注册表缓存来实现快速、增量的构建。它读取你的构建配置，将你的应用程序组织到不同的层（依赖项、资源、类）中，并只重新构建和推送发生变更的层。在项目进行快速迭代时，Jib 只讲发生变更的层（而不是整个应用程序）推送到注册表来节省宝贵的构建时间。
> 1. 可重现——Jib 支持根据 Maven 和 Gradle 的构建元数据进行声明式的容器镜像构建，因此，只要输入保持不变，就可以通过配置重复创建相同的镜像。


可以可以通过 `mvn clean compile jib:buildTar` 生成 `target/jib-image.tar` 然后用解压缩工具解压后进行分析，实际上jib会将lib中非快照部分放到一个层，将快照部分放到一个层，将源码编译后放到一个层。。。<br />
![snipaste20190208_211839.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1549631943594-f1682ff3-91f5-47e7-8af4-ba83490c44a0.png#align=left&display=inline&height=485&linkTarget=_blank&name=snipaste20190208_211839.png&originHeight=507&originWidth=780&size=83376&width=746#align=left&display=inline&height=485&originHeight=507&originWidth=780&width=746)



<a name="35808e79"></a>
## 参考资料

- [谷歌开源 Java 镜像构建工具 Jib](https://www.infoq.cn/article/2018%2F07%2Fgoogle-opensource-Jib)
- [jib自定义entrypoint](https://segmentfault.com/a/1190000016254151)
- [jib打包docker镜像实战](https://segmentfault.com/a/1190000016156009)
- [Jib - Containerize your Maven project](https://github.com/GoogleContainerTools/jib/blob/master/jib-maven-plugin/README.md)

