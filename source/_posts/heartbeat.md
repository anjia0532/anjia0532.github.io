---
title: ElkStack之Heartbeat(心跳检测)
date: 2017-03-03 01:20:16
tags: [elk,elkstasck,Heartbeat]
categories: [elkstasck]
---

Heartbeat是一个轻量级守护程序，安装在远程服务器上以定期检查服务的状态，检查服务是否可用。与Metricbeat不同，Metricbeat只告诉你服务器是启动还是停止，Heartbeat可以告诉你，服务是否可以正常访问

Heartbeat可以帮你验证服务是否可以正常访问，如果你需要验证内部服务时，他还可以用于其它方案，例如，安全用例

你可以配置Heartbeat来Ping指定主机名的所有DNS可解析的IP地址。从而检查所有的负载均衡服务，是否可用

配置Heartbeat时，可以指定用于监控的hosts 。 每个监控器按照你设置的监控计划运行。例如，你可以将一个监控器配置为每10分钟运行一次，并且配置不同的监控器在`9:00`~`17:00`运行

Heartbeat目前支持通过以下方式检查hosts
- ICMP(IPV4/IPV6)回显请求。当你只是想检查服务是否可用时，可以使用`icmp`。这个监控器需要管理员权限

- TCP。 `tcp`监控器是通过TCP协议来连接。可以选择配置`tcp`监控器，通过发送或接受自定义有效内容(payload)来验证端点(endpoint)是否可用

- HTTP。使用`http`监控器是通过http协议进行连接。可以选择配置`http`监控器来验证服务是否返回预期的响应，例如，特定的状态码，响应头或者内容

`tcp`和`http`都支持SSL/TLS和代理设置

# 安装Heartbeat

Heartbeat检测服务心跳，一般安装在较为稳定的独立服务器上（类似云服务，不断电，不断网）。尽量不要跟被监控的服务放在一个篮子里

从[下载页面][downloads]根据系统下载相应的安装包

**deb(Debian/Ubuntu)**
```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-5.2.2-amd64.deb
sudo dpkg -i heartbeat-5.2.2-amd64.deb
```

**rpm(Redhat / Centos / Fedora)**
```bash
curl -L O https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-5.2.2-x86_64.rpm
sudo rpm -vi heartbeat-5.2.2-x86_64.rpm
```

**mac**
```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-5.2.2-darwin-x86_64.tar.gz
tar xzvf heartbeat-5.2.2-darwin-x86_64.tar.gz
```

**windows**

1. 根据具体系统[下载][downloads] 32位系统 `https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-{version}-windows-x86.zip`或者 64位系统`https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-{version}-windows-x86_64.zip` 注意将`{version}`替换成具体版本,格式类似于`5.2.1`

1. 将下载的zip解压到指定文件夹，例如 `D:\Heartbeat`

1. 以管理员身份打开PowerShell(右键单击PowerShell图标，选择**以管理员身份运行**)。注意，如果是xp，需要单独安装powershell

1. 运行以下命令安装为Windows服务
```powershell
PS > cd 'D:\Heartbeat'
PS D:\Heartbeat> .\install-service-heartbeat.ps1
```

!> 如果脚本被禁用，或者安装不成功，或者是xp系统，其实可以考虑使用[nssm][nssm],具体用法，百度之。具体参数为`-c D:\Heartbeat\heartbeat.yml  -path.home D:\Heartbeat\ -path.data D:\Heartbeat\`

测试阶段可以使用 `heartbeat.exe -e -f heartbeat.yml`

如果已经安装服务，可以使用`net start heartbeat`(使用管理员权限的cmd或者powershell或者从服务(<kbd>Win</kbd>+<kbd>R</kbd>输入`services.msc`，找到heartbeat服务手动开启)


# 配置Heartbeat
可以通过编辑`heartbeat.yml`来配置heartbeat。`heartbeat.full.yml`里面有所有可用的选项，可以作为参考

Heartbeat提供在指定的间隔时间检测主机心跳状态的监控，可以单独配置每个监控。Heartbeat目前提供ICMP,TCP 和HTTP 的监控（更多有关监控的信息，参见 [简介][overview]）

#### 要启用的监控列表，使用(`-`) 开头(yaml中的数组),以下表示的用Heartbeat监控`ICMP`和`TCP`

```yaml
heartbeat.monitors:
- type: icmp
  schedule: '*/5 * * * * * *'   #1
  hosts: ["myhost"]
- type: tcp
  schedule: '@every 5s'         #2
  hosts: ["myhost:7"]  # default TCP Echo Protocol
  mode: any                     #3
  check.send: "Check"
  check.receive: "Check"
- type: http
  schedule: '@every 5s'
  urls: ["http://localhost:80/service/status"]
  check.response.status: 200
heartbeat.scheduler:
  limit: 10
```

1. 这个ICMP监控，每五秒钟运行一次(e.g. `10:00:00`,`10:00:05` ...) `schedule`选项是类`cron`语法。具体参见[this cronexpr implementation][cronexpr]

2. 这个TCP监控也是每5秒运行一次。Heartbeat添加了`@every`关键词添加到了`conexpr`包里

3. `mode`指定是否用来ping一个ip（`any`）或全解析IPS(`all`) 。

[原版配置][heartbeat-configuration-details]

#### 监控选项

###### type
- `icmp`(IPV4/IPV6)回显请求。当你只是想检查服务是否可用时，可以使用`icmp`。这个监控器需要管理员权限

- `tcp`。 `tcp`监控器是通过TCP协议来连接。可以选择配置`tcp`监控器，通过发送或接受自定义有效内容(payload)来验证端点(endpoint)是否可用

- `http`。使用`http`监控器是通过http协议进行连接。可以选择配置`http`监控器来验证服务是否返回预期的响应，例如，特定的状态码，响应头或者内容

`tcp`和`http`都支持SSL/TLS和代理设置
###### name
监控器名字

###### enabled
Boolean值，指定监控模块是否启用，默认为true

###### schedule
类cron表达式

###### ipv4
Boolean值，如果指定了host，是否使用ipv4协议进行pin，默认为true

###### ipv6
Boolean值，如果指定了host，是否使用ipv4协议进行pin，默认为true

###### mode
`any`或者`all`,默认为`any`。如果是`any`，监控器对指定的主机名只ping一个ip地址。如果是`all`，则ping所有dns能解析出来的ip地址。对于负载均衡监控很有用

###### watch.poll_file
!> 此为实验功能。未来可能更改或删除

这是JSON格式的监控器配置文件。可以包含多个需要监控的对象。Heartbeat定期检查此文件。Heartbeat会合并heartbeat.yml和json中的配置，有新增的则新增监控实例。josn文件中删除实例后，heartbeat会停止监控该实例。

每个监控器用协议，主机，端口等参数作为唯一id。如果存在相同的，则使用合并后的最后一个json定义的设置。(以json中定义的为准)。所以为了不重启heartbeat，建议使用`watch.poll_file`进行配置，但是需要注意，这个是实验室功能，后期可能会修改或者变更

```yaml
heartbeat.monitors:
- type: tcp
  schedule: '*/5 * * * * * *'
  hosts: ["myhost"]
  watch.poll_file:
    path: {path.config}/monitors/dynamic.json
    interval: 5s
```
    path

        指定的JSON文件地址

    interval

        指定间隔时间

JSON文件内容如下
```javascript
{"hosts": ["myhost:1234"], "schedule": "*/15 * * * * * *"}     #1
{"hosts": ["tls://otherhost:479"], "ssl.certificate_authorities": ["path/to/ca/file.pem"]}      #2
```

1. 检查到文件变更后，heartbeat会重启该监控器，并改为每15秒钟运行一次
2. heartbeat新增一个监控，使用带有ca证书的基于TLS的连接

##### ICMP选项

`type`设置为`icmp`时，该项生效。Heartbeat使用ICMP(v4和v6)回显请求来检查配置的主机

###### hosts
需要ping的主机列表

###### wait
等待时间，默认1s

##### TCP 选项
`type`设置为`tcp`时，该项生效。通过tcp协议发送或接受自定义内容来验证端点是否可用。

###### hosts
需要ping的主机列表。
* 简单的主机名，例如`localhost` 或者ip地址。如果你指定了这个选项，你必须在指定`ports`选项。如果监控器配置了使用ssl，heartbeat使用基于ssl、tls的连接。否则的话，使用普通的tcp连接
* 主机名+端口，例如`localhost:8080`。heartbeat根据主机名和端口号进行连接。如果监控器配置了使用ssl，heartbeat使用基于ssl、tls的连接。否则的话，使用普通的tcp连接
* 完整的URL，语法为 `scheme://<host>:[port]`
    - `scheme` 为 `tcp`,`plain`,`ssl`或者`tls`。如果指定的是`tcp`或者`plain`，heartbeat使用tcp连接即使监控器配置为使用ssl，如果指定了`tls`或者`ssl`,heartbeat建立ssl连接。但是如果监控器没用ssl，则使用系统默认值(暂不支持windows)
    - `host`是主机名。
    - `port`是端口号。

###### ports
如果`hosts`中没指定端口，则在此需要配置需要ping的端口列表。例如检查 80,9200,5044端口
```yaml
- type: tcp
  schedule: '@every 5s'
  hosts: ["myhost"]
  ports: [80, 9200, 5044]
```

###### check
验证发送到主机的有效内容(payload)和预期的响应。如果未指定有效内容(payload)，一旦连接成功，则视为可用。如果只指定了发送，未指定接收。接收到任何响应都视为成功。如果只指定接收内容，未指定发送内容。不发送payload，但是在连接中，客户端希望接收到的内容为`hello message`或者`banner`(原文: If receive is specified without send, no payload is sent, but the client expects to receive a payload in the form of a "hello message" or "banner" on connect.)
```yaml
- type: tcp
  schedule: '@every 5s'
  hosts: ["myhost"]
  ports: [7]
  check.send: 'Hello World'
  check.receive: 'Hello World'
```
###### proxy_url
只可以用socks5代理。
```yaml
proxy_url: socks5://user:password@socks5-proxy:2233
```
使用代理时，主机名实在代理服务器上解析，而不是在客户端解析。可以通过设置 proxy_use_local_resolver来修改

###### proxy_use_local_resolver
Boolean值，用于确定主机名是否本地解析还是在代理服务器解析。默认值为false，即在代理服务器解析。

###### ssl
TLS/SSL连接设置。如果`check`未配置，则监控器将仅检查是否可以建立SSL/TLS连接。此检查可能在TCP级别或在证书验证期间失败

```yaml
- type: tcp
  schedule: '@every 5s'
  hosts: ["myhost"]
  ports: [80, 9200, 5044]
  ssl:
    certificate_authorities: ['/etc/ca.crt']
    supported_protocols: ["TLSv1.0", "TLSv1.1", "TLSv1.2"]
```

##### HTTP选项
`type`设置为`http`时，该项生效。通过http协议验证host是否返回预期响应。

###### urls
用于连接的URLs列表

```yaml
- type: http
  schedule: '@every 5s'
  urls: ["http://myhost:80"]
```
###### proxy_url
http代理url。选填项。如果不设置，默认使用系统环境中的`HTTP_PROXY`

###### username
选填项。用来请求身份验证的服务。如果验证身份的服务不指定，很可能返回403

###### password
选填项。同username

###### ssl 同tcp ssl

###### check(咳咳，划重点)

选填项。发送`request`到远程服务，并接受期望响应`response`

```yaml
- type: http
  schedule: '@every 5s'
  urls: ["http://myhost:80"]
  check.request.method: HEAD
  check.response.status: 200
```

* `check.request` 选项
    - method - HTTP方法。支持`HEAD`,`GET`和`POST`
    - headers - 设置请求头
    - body - 选填请求体(用于POST方法)

* `check.response` 选项
    - status - 期望的响应码。未设置或者设置的是`0`，除`404`以外状态码均可
    - headers - 必须响应的header头信息
    - body - 必须的响应体

```yaml
- type: http
  schedule: '@every 5s'
  urls: ["https://myhost:80"]
check.request:
  method: GET
  headers:
    'X-API-Key': '12345-mykey-67890'
check.response:
  status: 200
  body: '{"status": "ok"}'
```

##### Scheduler 选项

```yaml
heartbeat.scheduler:
  limit: 10
  location: 'UTC-08:00'
```
示例中设置`limit`为10，确保只有10个IO任务处于活动状态。IO任务可以是通过DNS实际检查或者解析地址

###### limit
允许Heartbeat执行的并发IO任务数。如果为0，则没有限制。默认值为0。大多数操作系统文件，将文件描述符限制设置为1024。为了Heartbeat正确运行并且不意外组织输出。应该将`limit`的值设置低于`ulimit`

###### location
设置时区。默认使用本地实际 `localtime`

#### 发送到Elasticsearch
```yaml
output.elasticsearch:
  hosts: ["192.168.1.42:9200"]
  template.name: "heartbeat"                #1
  template.path: "heartbeat.template.json"  #2
```
1,2处是自动在Elasticsearch中加载索引模板，详细信息[参见官网文档][heartbeat-template]

如果是要输出到Logstash，参见[配置Heartbeat使用Logstash][config-heartbeat-logstash]

!> 如果要测试配置，在heartbeat可执行目录下，运行`./heartbeat -configtest -e`

# 运行Heartbeat

deb :
```bash
sudo /etc/init.d/ start
```

rpm :
```bash
sudo /etc/init.d/heartbeat start
```

mac :
```bash
sudo ./heartbeat -e -c heartbeat.yml -d "publish"
```

win : **管理员权限**
```bash
net start heartbeat
```
Windows默认将log输出在`${Heartbeat_home}\Logs`文件夹

?> 目前为止，Heartbeat已经开始检查你的服务状态并且发送相应的数据到你定义的输出点了(logstash/elasticsearch)

# 命令行选项

?> 命令行运行`./heartbeat -h`查看完整的选项列表

`-E <setting>=<value>`

    覆盖配置文件中的某个配置例如 `./heartbeat -c heartbeat.yml -E name=mybeat`

`-N`

    禁止发送数据到指定的输出。这个选项在测试Beat时很有用

`-c <file>`

    指定heartbeat配置文件

`configtest`

    测试配置文件是否可用，然后退出。在排除配置文件错误时很有用

`-cpuprofile <output file>`

    将cpu配置信息输出到指定文件。在排除故障的时候很有用

`-d <selectors>`

    使用指定的选择器进行调试。参数用逗号隔开，或者使用 `-d "*"`调试所有的组件。例如`-d "publish"`显示所有`"publish"`相关的信息

`-e`

    禁用syslog/file输出，只记录到stderr

`-httpprof [<host>]:<port>`

    启动http服务器进行性能分析

`-memprofile <output file>`

    将内存配置信息写入到指定文件。

`-path.config`

    设置配置文件的路径

`-path.data`

    设置data文件路径

`-path.home`

    设置可执行文件所在路径

`-path.logs`

    设置日志文件的路径

`-v`

    启用详细输出，以显示INFO级别日志

`-version`

    显示beat版本并退出


本文只是针对官网文档进行了部分翻译。其他像是[输出到logstash,redis等配置信息][configuring-howto-heartbeat]以及[Processors][configuration-processors]部分[Exported Fields][exported-fields]部分,[Securing Heartbeat][securing-heartbeat]暂不翻译

[downloads]: https://www.elastic.co/downloads/beats/heartbeat
[nssm]: http://nssm.cc/download
[overview]: https://www.elastic.co/guide/en/beats/heartbeat/current/heartbeat-overview.html
[cronexpr]: https://github.com/gorhill/cronexpr#implementation
[heartbeat-configuration-details]: https://www.elastic.co/guide/en/beats/heartbeat/current/heartbeat-configuration-details.html
[config-heartbeat-logstash]: https://www.elastic.co/guide/en/beats/heartbeat/current/config-heartbeat-logstash.html
[heartbeat-template]: https://www.elastic.co/guide/en/beats/heartbeat/current/heartbeat-template.html
[configuring-howto-heartbeat]: https://www.elastic.co/guide/en/beats/heartbeat/current/configuring-howto-heartbeat.html
[exported-fields]: https://www.elastic.co/guide/en/beats/heartbeat/current/exported-fields.html
[securing-heartbeat]: https://www.elastic.co/guide/en/beats/heartbeat/current/securing-heartbeat.html
[configuration-processors]: https://www.elastic.co/guide/en/beats/heartbeat/current/configuration-processors.html
