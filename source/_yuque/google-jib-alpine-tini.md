
---

title: 046-解决google jib多任务问题

date: 2019-09-22 23:30:01 +0800

tags: [java,docker,google-jib]

categories: java,docker

---

> 这是坚持技术写作计划（含翻译）的第46篇，定个小目标999，每周最少2篇。


本文主要讲解使用google jib打包应用时 如果 基础镜像是 openjdk 的alpine系列，运行成功后无法正常使用 jmap,jstack,arthas 等工具，报错信息如下
```bash
[ERROR] Start arthas failed, exception stack trace:
com.sun.tools.attach.AttachNotSupportedException: Unable to get pid of LinuxThreads manager thread
at sun.tools.attach.LinuxVirtualMachine.(LinuxVirtualMachine.java:86)
at sun.tools.attach.LinuxAttachProvider.attachVirtualMachine(LinuxAttachProvider.java:78)
at com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:250)
at com.taobao.arthas.core.Arthas.attachAgent(Arthas.java:72)
at com.taobao.arthas.core.Arthas.(Arthas.java:25)
at com.taobao.arthas.core.Arthas.main(Arthas.java:99)
[ERROR] attach fail, targetPid: 1
```

在之前写的 [使用Maven加速和简化构建Docker(基于Google jib)](https://juejin.im/post/5c60c021f265da2dd37bf85b) 最后有提过，但是没有详细展开。

google jib 和 Dragonfly 系列文章

- [使用Maven加速和简化构建Docker(基于Google jib)](https://juejin.im/post/5c60c021f265da2dd37bf85b)
- [012-P2P加速Docker镜像分发(阿里Dragonfly)](https://juejin.im/post/5c98a8e9f265da60e346fe04)
- [013-阿里Dragonfly体验之私有registry下载](https://juejin.im/post/5c9edf58f265da3094115623)
- [046-解决google jib多任务问题](http://juejin.im/post/5d889a2bf265da03c9273875)

<!-- more -->

<a name="eyQ52"></a>
## 简述 google jib 和阿里 Dragonfly

阿里的dragonfly和google jib其实都在解决类似的问题，如何最大化的重用依赖。但是是两个思路，dragonfly是在不改动源镜像的基础上，类似cdn的思路在近源缓存数据，这样一来局域网内就可以不用重复从公网拉取镜像，同时因为使用了p2p传输，所以可以有效防止单机热点打满带宽的问题。(d7y主要是使用p2p解决大文件传输问题，用于镜像传输只是捎带手的一个功能)，d7y是解决制成品如何优化传输。

google jib的思路是，侵入到构建过程中，发挥主人翁精神，我构建，我优化。比如在使用gradle，maven构建时，体积最大的是jre或者jdk,其次是各种三方jar包，再次之的是web中的静态资源(那些往resource可劲塞文件的不在讨论范围内)，而这些东西往往又不会经常变动，而源码部分会经常变动，那么将其分离开，利用docker的layer进行缓存，既能节省空间，又能节省带宽，又能节省时间。

<a name="nEDx9"></a>
## 配置google jib
<a name="45HSV"></a>
### settings.xml 增加auth配置
settings.xml 增加registry server 的auth认证，password可以用 [maven password encryption](https://maven.apache.org/guides/mini/guide-encryption.html) ,不建议存放明文密码

```xml
<settings>
  ...
  <servers>
    ...
    <server>
      <id>MY_REGISTRY</id>
      <username>MY_USERNAME</username>
      <password>{MY_SECRET}</password>
    </server>
  </servers>
</settings>
```

<a name="i25Wx"></a>
### anjia0532/openjdk-8-alpine-lib的Dockerfile

```xml
FROM openjdk:8-jdk-alpine

ARG ARTHAS_VERSION="3.1.3"
ARG MIRROR=false

ENV MAVEN_HOST=http://repo1.maven.org/maven2 \
    ALPINE_HOST=dl-cdn.alpinelinux.org \
    MIRROR_MAVEN_HOST=http://maven.aliyun.com/repository/public \
    MIRROR_ALPINE_HOST=mirrors.tuna.tsinghua.edu.cn

# if use mirror change to aliyun mirror site
RUN if $MIRROR; then MAVEN_HOST=${MIRROR_MAVEN_HOST} ;ALPINE_HOST=${MIRROR_ALPINE_HOST} ; sed -i "s/dl-cdn.alpinelinux.org/${ALPINE_HOST}/g" /etc/apk/repositories ; fi

RUN \
    # https://github.com/docker-library/openjdk/issues/76
    apk add --no-cache tini

RUN \
    # change to GMT+0800 timezone
    apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" >  /etc/timezone

RUN  \
    # download & install arthas
    wget -qO /tmp/arthas.zip "${MAVEN_HOST}/com/taobao/arthas/arthas-packaging/${ARTHAS_VERSION}/arthas-packaging-${ARTHAS_VERSION}-bin.zip" && \
    mkdir -p /opt/arthas && \
    unzip /tmp/arthas.zip -d /opt/arthas && \
    rm /tmp/arthas.zip

# Tini is now available at /sbin/tini
ENTRYPOINT ["/sbin/tini", "--"]
```

构建命令
```bash
docker build --build-arg MIRROR=true . -t anjia0532/openjdk-8-alpine-lib
```

<a name="K5WJF"></a>
### 修改pom.xml
```xml
<project>
    <groupId>com.example.module</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>demo</name>
    ...
    <build>
        <plugins>
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>1.6.1</version>
                <configuration>
                    <from>
                        <image>anjia0532/openjdk-8-alpine-lib</image>
                    </from>
                    <to>
                        <image>${project.artifactId}:${project.version}</image>
                    </to>
                    <container>
                        <entrypoint>
                            <shell>/sbin/tini</shell>
                            <option>--</option>
                        </entrypoint>
                        <args>
                            <arg>java</arg>
                            <arg>-cp</arg>
                            <arg>/app/resources/:/app/classes/:/app/libs/*</arg>
                            <arg>com.example.application</arg>
                        </args>
                        <ports>
                            <port>8080</port>
                            <port>8563</port><!-- arthas 端口 -->
                        </ports>
                        <environment>
                            <_JAVA_OPTIONS>-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/</_JAVA_OPTIONS>
                        </environment>
                    </container>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

主要起作用的是 tini ,需要在entrypoint部分设置 `/sbin/tini --` 但是 根据文档 [https://github.com/GoogleContainerTools/jib/blob/master/docs/faq.md#what-would-a-dockerfile-for-a-jib-built-image-look-like](https://github.com/GoogleContainerTools/jib/blob/master/docs/faq.md#what-would-a-dockerfile-for-a-jib-built-image-look-like) 所述

```xml
ENTRYPOINT ["java", jib.container.jvmFlags, "-cp", "/app/resources:/app/classes:/app/libs/*", jib.container.mainClass]
CMD [jib.container.args]
```

jib构建的应用，默认会使用 将启动命令放到entrypoint 部分，会覆盖掉baseimage里写的entryoint,如果在pom里定义的话，又会覆盖jib的默认启动命令，所以只能将启动项目放到 args 部分，但是又无法使用jvmFlags，默认cp，默认main类推断，必须要显式的自行声明。

<a name="kldjO"></a>
### 生成镜像

```bash
mvn compile -Pprod jib:build  # 构建并推送到目标registry上，构建主机不需要安装docker
mvn compile -Pprod jib:dockerBuild # 构建到本机docker daemon
mvn compile -Pprod jib:buildTar # 构建并生成tar.gz,可以使用docker load --input target/jib-image.tar 导入
```

如果是在jenkins等CICD场景，每次让研发修改pom不太方便，可以使用命令行来构建。

```bash
mvn compile com.google.cloud.tools:jib-maven-plugin:1.6.1:dockerBuild \
    -Djib.to.image=myregistry/myimage:latest \
    -Djib.to.auth.username=$USERNAME \
    -Djib.to.auth.password=$PASSWORD \
    -Djib.container.environment=key1="value1",key2="value2" \
    -Djib.container.args=arg1,arg2,arg3
```

<a name="yIypH"></a>
## 另外
上述示例不管是maven还是gradle都是java类应用，对于nodejs或者golang，jib官方也支持用户基于[jib core](https://github.com/GoogleContainerTools/jib/tree/master/jib-core)自行编写构建插件

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。<br />长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

<a name="35808e79"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/09/22/google-jib-alpine-tini/)
- [我的掘金](http://juejin.im/post/5d889a2bf265da03c9273875)
- [Jib - Containerize your Maven project](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin)
- [jib core](https://github.com/GoogleContainerTools/jib/tree/master/jib-core)

