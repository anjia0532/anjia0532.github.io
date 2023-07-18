---
title: 075-Dlink+Flink+CDC on k8s 整库同步
urlname: dlink-flink-flink-cdc-on-k8s
date: '2022-08-02 19:35:21 +0800'
tags:
  - 大数据
  - hadoop
  - flink
categories:
  - flink
---

> 这是坚持技术写作计划（含翻译）的第 75 篇，定个小目标 999，每周最少 2 篇。

本文 主要讲解如何在 K8S 集群跑 Dlink+Flink 通过 Flink CDC 进行整库同步。

<!-- more -->

## 安装 K8S

如果是本地测试的话，可以起 [minikube](https://minikube.sigs.k8s.io/docs/start/)

如果是生产的话，可以使用 [Rancher RKE2](https://docs.rke2.io/install/quickstart/) 配合 [Rancher ](https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/)使用，也可以用别的商用 K8S 方案(比如 阿里云 ACK，腾讯云 TKE 等）

## 安装 Flink Operator 及安装 Flink Session 到 K8S 集群

### 安装 Flink Operator 到 K8S 集群

本文假设你已经会一些最基本的 k8s 知识，比如

- k8s 集群配置了加速器(flink,dlink,doris,hadoop 相关镜像没一个小的，不用加速器，得慢死)
- kubectl 的使用，`~/.kube/config`,
- 如何安装 `helm`,

如果不会，请自行搜索相关知识。

```bash
# 安装 cert-manager 必选
kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml

# 可以将 flink-kubernetes-operator-1.1.0 换成别的版本，具体以 https://downloads.apache.org/flink/ 列出为准
helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.1.0/

# 安装 flink-kubernetes-operator 到 k8s 集群 (--namespace 可以缩写 -n,不写默认装到 default 集群，如果命名空间不存在，可以加上 --create-namespace ，在安装时创建命名空间)
helm install flink-kubernetes-operator --create-namespace --namespace flink flink-operator-repo/flink-kubernetes-operator --set image.repository=apache/flink-kubernetes-operator
```

参考 [flink-kubernetes-operator quick-start](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/try-flink-kubernetes-operator/quick-start/)

### 安装 Flink Session 集群到 K8S 集群

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: flink-session
spec:
  # 官方镜像少 jar ,我自己打的镜像，只用于演示本文，实际生产请自行构建镜像
  # Flink CDC 目前只支持 flink 1.14.* ,暂不支持 1.15.*
  image: anjia0532/flink:1.14.5-scala_2.12-java8-5
  # Flink 版本改成 1.14.*
  flinkVersion: v1_14
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "2"
  serviceAccount: flink
  jobManager:
    resource:
      memory: "2048m"
      cpu: 1
  taskManager:
    resource:
      memory: "2048m"
      cpu: 1
```

Flink CDC 目前只支持 flink 1.14._ ,暂不支持 1.15._

参考 [flink-cdc-connectors 支持的 Flink 版本](https://ververica.github.io/flink-cdc-connectors/master/content/about.html#supported-flink-versions)

```bash
# 安装 到 flink 命名空间
kubectl -n flink apply -f flink-session-only.yaml
```

会自动创建 Flink Session Deployment（部署） 和 对应的 Service (服务发现 )
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1659495131146-0275be85-3b41-40ec-b558-ca51ed4cf7a9.png#clientId=uac603524-b4d3-4&from=paste&height=476&id=u8f07ab33&originHeight=476&originWidth=1674&originalType=binary∶=1&rotation=0&showTitle=false&size=64468&status=done&style=none&taskId=u0d814ac9-9744-4802-99a8-5ff9d5294e0&title=&width=1674)

参考 [flink-kubernetes-operator examples](https://github.com/apache/flink-kubernetes-operator/tree/main/examples)
[
](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/try-flink-kubernetes-operator/quick-start/)

## 安装 Dlink 到 K8S 集群

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: flink-dlink
  name: dlink
spec:
  selector:
    matchLabels:
      app: flink-dlink
  template:
    metadata:
      labels:
        app: flink-dlink
    spec:
      containers:
        - image: anjia0532/dlink:v0.6.6-1
          name: dlink
          volumeMounts:
            - mountPath: /opt/dlink/config/application.yml
              name: admin-config
              subPath: application.yml
      volumes:
        - configMap:
            name: dlink-config
          name: admin-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: dlink-config
data:
  application.yml: |-
    spring:
      datasource:
        url: jdbc:mysql://mysql-headless.mysql:3306/dlink?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false&zeroDateTimeBehavior=convertToNull&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
        username: dlink
        password: dlink
        driver-class-name: com.mysql.cj.jdbc.Driver
      application:
        name: dlink
    #  flyway:
    #    enabled: false
    #    clean-disabled: true
    ##    baseline-on-migrate: true
    #    table: dlink_schema_history
      # Redis配置
      #sa-token如需依赖redis，请打开redis配置和pom.xml、dlink-admin/pom.xml中依赖
      # redis:
      #   host: localhost
      #   port: 6379
      #   password:
      #   database: 10
      #   jedis:
      #     pool:
      #       # 连接池最大连接数（使用负值表示没有限制）
      #       max-active: 50
      #       # 连接池最大阻塞等待时间（使用负值表示没有限制）
      #       max-wait: 3000
      #       # 连接池中的最大空闲连接数
      #       max-idle: 20
      #       # 连接池中的最小空闲连接数
      #       min-idle: 5
      #   # 连接超时时间（毫秒）
      #   timeout: 5000
    server:
      port: 8888

    mybatis-plus:
      mapper-locations: classpath:/mapper/*Mapper.xml
      #实体扫描，多个package用逗号或者分号分隔
      typeAliasesPackage: com.dlink.model
      global-config:
        db-config:
          id-type: auto
      configuration:
      ##### mybatis-plus打印完整sql(只适用于开发环境)
    #    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
        log-impl: org.apache.ibatis.logging.nologging.NoLoggingImpl

    # Sa-Token 配置
    sa-token:
        # token名称 (同时也是cookie名称)
        token-name: satoken
        # token有效期，单位s 默认10小时, -1代表永不过期
        timeout: 36000
        # token临时有效期 (指定时间内无操作就视为token过期) 单位: 秒
        activity-timeout: -1
        # 是否允许同一账号并发登录 (为true时允许一起登录, 为false时新登录挤掉旧登录)
        is-concurrent: false
        # 在多人登录同一账号时，是否共用一个token (为true时所有登录共用一个token, 为false时每次登录新建一个token)
        is-share: true
        # token风格
        token-style: uuid
        # 是否输出操作日志
        is-log: false
---
apiVersion: v1
kind: Service
metadata:
  name: flink-dlink
spec:
  ipFamilies:
    - IPv4
  ipFamilyPolicy: SingleStack
  ports:
    - name: http
      port: 8888
      protocol: TCP
      targetPort: 8888
  selector:
    app: dlink
  type: ClusterIP
```

注意修改 ConfigMap 里的 dlink 链接的 MySQL 的地址，用户名，密码，以及执行
[https://github.com/DataLinkDC/dlink/tree/dev/dlink-doc/sql](https://github.com/DataLinkDC/dlink/tree/dev/dlink-doc/sql) 里的 [dlink.sql](https://github.com/DataLinkDC/dlink/blob/dev/dlink-doc/sql/dlink.sql) (第一次执行) 和 [dlinkmysqlcatalog.sql](https://github.com/DataLinkDC/dlink/blob/dev/dlink-doc/sql/dlinkmysqlcatalog.sql)（第一次执行），如果是已经存在了，只是要升级,执行 [dlink_history.sql](https://github.com/DataLinkDC/dlink/blob/dev/dlink-doc/sql/dlink_history.sql)

```bash
kubectl -n flink apply -f dlink.yaml
```

可以使用 Idea 里的 [Kubernates](https://plugins.jetbrains.com/plugin/10485-kubernetes), [Nocalhost](https://plugins.jetbrains.com/plugin/16058-nocalhost), 或者 VS Code 里的 [Kubernates](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools), [Nocalhost ](https://marketplace.visualstudio.com/items?itemName=nocalhost.nocalhost)或者命令行程序 [k9s ](https://github.com/derailed/k9s)或者[kubectl](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) 把 dlink 和 flink job UI 的端口转出来。

## 整库同步

需要确保 dlink 所在的 MySQL 的 my.conf 开了 binglog ,并且 格式为 ROW
演示 整库同步 dlink 的所有表同步到 cdc-test 中。
在 dlink 的 MySQL 数据库中创建一个名为 cdc-test 的库。

```sql
EXECUTE CDCSOURCE cdc_mysql1 WITH (
  'connector' = 'mysql-cdc',
  'hostname' = 'mysql-headless.mysql',
  'port' = '3306',
  'username' = 'dlink',
  'password' = 'dlink',
  'checkpoint' = '3000',
  'scan.startup.mode' = 'initial',
  'parallelism' = '1',
  'table-name' = 'dlink\..*',
  'sink.url' = 'jdbc:mysql://mysql-headless.mysql:3306/cdc-test?characterEncoding=utf-8&useSSL=false',
  'sink.username' = 'dlink',
  'sink.password' = 'dlink',
  'sink.connector' = 'jdbc',
  'sink.sink.db' = 'cdc-test',
  'sink.table.prefix' = '',
  'sink.table.lower' = 'true',
  'sink.table-name' = '${tableName}',
  'sink.driver' = 'com.mysql.jdbc.Driver',
  'sink.sink.buffer-flush.interval' = '2s',
  'sink.sink.buffer-flush.max-rows' = '100',
  'sink.sink.max-retries' = '5'
)
```

打开 Dlink Web 添加 K8S Session 集群，添加 作业，复制 CDCSource SQL 保存，并执行、或者异步提交。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1659497355190-ae8dd3c5-a68b-4edd-a86e-e0671acfb27a.png#clientId=uac603524-b4d3-4&from=paste&height=885&id=u998e7d68&originHeight=885&originWidth=1896&originalType=binary∶=1&rotation=0&showTitle=false&size=155309&status=done&style=none&taskId=uac191e3e-166c-47f5-8bb2-239f6096365&title=&width=1896)
打开 Flink UI 看看执行情况。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1659498243983-5d07126a-471d-4d6e-8f50-fdba428f82d5.png#clientId=uac603524-b4d3-4&from=paste&height=750&id=u71881d28&originHeight=750&originWidth=1565&originalType=binary∶=1&rotation=0&showTitle=false&size=102377&status=done&style=none&taskId=u65703133-9d57-4c4c-8867-32e51080649&title=&width=1565)

## 附录

### Dlink docker file

```dockerfile
FROM flink:1.14.5-scala_2.12-java8 as builder

FROM openjdk:8-jdk

ARG DLINK_VERSION="0.6.6"

WORKDIR /opt/dlink

ADD https://github.com/DataLinkDC/dlink/releases/download/v${DLINK_VERSION}/dlink-release-${DLINK_VERSION}.tar.gz /tmp/dlink.tar.gz

#COPY ./dlink-release-${DLINK_VERSION}.tar.gz /tmp/dlink.tar.gz

RUN tar zxf /tmp/dlink.tar.gz -C /opt/dlink --strip-components=1 && mkdir -p /opt/dlink/plugins/ && rm -rf /tmp/*

ADD https://maven.aliyun.com/repository/central/ru/yandex/clickhouse/clickhouse-jdbc/0.2.6/clickhouse-jdbc-0.2.6.jar /opt/dlink/plugins/
ADD https://maven.aliyun.com/repository/central/mysql/mysql-connector-java/8.0.22/mysql-connector-java-8.0.22.jar /opt/dlink/plugins/
ADD https://maven.aliyun.com/repository/central/com/ververica/flink-sql-connector-mysql-cdc/2.2.1/flink-sql-connector-mysql-cdc-2.2.1.jar /opt/dlink/plugins/
ADD https://maven.aliyun.com/repository/central/org/apache/flink/flink-connector-jdbc_2.12/1.14.5/flink-connector-jdbc_2.12-1.14.5.jar /opt/dlink/lib/

#ADD ./flink-shaded-hadoop-3-uber-3.1.1.7.2.9.0-173-9.0.jar /opt/dlink/plugins/

COPY --from=builder /opt/flink/lib/* /opt/dlink/plugins/

RUN cp /opt/dlink/extends/dlink-client-1.14-0.6.6.jar /opt/dlink/lib/
RUN cp /opt/dlink/extends/dlink-catalog-mysql-1.14-0.6.6.jar /opt/dlink/lib/
RUN cp /opt/dlink/extends/dlink-connector-jdbc-1.14-0.6.6.jar /opt/dlink/lib/


RUN rm -rf /opt/dlink/lib/dlink-client-1.13-0.6.6.jar && rm -rf /opt/dlink/lib/dlink-catalog-mysql-1.13-0.6.6.jar && rm -rf /opt/dlink/lib/dlink-connector-jdbc-1.13-0.6.6.jar

CMD [ "/bin/sh", "-c", "java -Dloader.path=./lib,./plugins -Ddruid.mysql.usePingMethod=false -jar -Xms512M -Xmx2048M ./dlink-admin-*.jar" ]
```

```bash
docker build . -f Dockerfile-dlink --build-arg DLINK_VERSION="0.6.6" -t anjia0532/dlink:v0.6.6-1
docker push anjia0532/dlink:v0.6.6-1
```

### Flink Docker file

```dockerfile
FROM anjia0532/dlink:v0.6.6-1 as builder

FROM flink:1.14.5-scala_2.12-java8

COPY --from=builder /opt/dlink/lib/dlink-client-1.14-0.6.6.jar /opt/flink/lib/
COPY --from=builder /opt/dlink/jar/dlink-client-base-0.6.6.jar /opt/flink/lib/
COPY --from=builder /opt/dlink/jar/dlink-common-0.6.6.jar /opt/flink/lib/
COPY --from=builder /opt/dlink/lib/dlink-catalog-mysql-1.14-0.6.6.jar /opt/flink/lib/

COPY --from=builder /opt/dlink/plugins/flink-sql-connector-mysql-cdc-2.2.1.jar /opt/flink/lib/

ADD https://maven.aliyun.com/repository/central/org/apache/flink/flink-connector-jdbc_2.12/1.14.5/flink-connector-jdbc_2.12-1.14.5.jar /opt/flink/lib/

RUN chown -R flink:flink /opt/flink/

ENTRYPOINT ["/docker-entrypoint.sh"]
EXPOSE 6123 8081
CMD ["help"]
```

```bash
docker build . -f Dockerfile-flink -t anjia0532/flink:1.14.5-scala_2.12-java8-1
docker push anjia0532/flink:1.14.5-scala_2.12-java8-1
```

### MySQL 表名查询 SQL

```sql
SELECT GROUP_CONCAT(CONCAT(table_schema,"\\.",table_name))
from `information_schema`.`TABLES` WHERE table_schema='dlink';

## 结果为 dlink\.dlink_alert_group,dlink\.dlink_alert_history,dlink\.dlink_alert_instance,dlink\.dlink_catalogue,dlink\.dlink_cluster,dlink\.dlink_cluster_configuration,dlink\.dlink_database,dlink\.dlink_flink_document,dlink\.dlink_history,dlink\.dlink_jar,dlink\.dlink_job_history,dlink\.dlink_job_instance,dlink\.dlink_savepoints,dlink\.dlink_schema_history,dlink\.dlink_sys_config,dlink\.dlink_task,dlink\.dlink_task_statement,dlink\.dlink_task_version,dlink\.dlink_user,dlink\.metadata_column,dlink\.metadata_database,dlink\.metadata_database_property,dlink\.metadata_function,dlink\.metadata_table,dlink\.metadata_table_property


SELECT GROUP_CONCAT( DISTINCT CONCAT(table_schema,"\\..*") ORDER BY table_schema )
from `information_schema`.`TABLES`;

## 结果为 canal_manager\..*,cdc-test\..*,datax_web\..*,dlink\..*,information_schema\..*,mysql\..*,performance_schema\..*,sys\..*
```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/08/02/dlink-flink-flink-cdc-on-k8s/)
- [我的掘金](https://juejin.cn/post/7127518515686277151/)
- [FlinkCDC 整库实时入仓入湖\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1Fr4y1j7Ku)
- [Dinky 实践系列之 FlinkCDC 整库实时入仓入湖](https://mp.weixin.qq.com/s/cr6YoRdU51dr0bjUVP3wQw)
- [CDCSOURCE 整库同步](http://www.dlink.top/docs/data_integration_guide/cdcsource_statements)
- [Flink SQL 作业快速入门](http://www.dlink.top/docs/quick_start/flinksql_quick_start)
- [基于 Flink CDC 同步 MySQL 分库分表构建实时数据湖](https://ververica.github.io/flink-cdc-connectors/master/content/%E5%BF%AB%E9%80%9F%E4%B8%8A%E6%89%8B/build-real-time-data-lake-tutorial-zh.html)
