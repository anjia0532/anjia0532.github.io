---
title: 015-Ansible批量安装Elastic Beats(支持Linux和Windows)
urlname: ansible-beats
date: '2019-04-13 20:41:00 +0800'
tags:
  - ansible
  - linux
  - 运维
  - beats
  - es
  - elasticsearch
categories: '运维,elasticsearch'
---

> 这是坚持技术写作计划（含翻译）的第 15 篇，定个小目标 999，每周最少 2 篇。

使用[elastic beats](https://www.elastic.co/cn/downloads/beats)进行拨测，metric 采集，主机监控，但是批量化安装仍是个问题，好在 elastic 官方有开源的  [ansible-beats](https://github.com/elastic/ansible-beats)  但是只支持 Linux，而我们在某些业务场景下，还有 WinServer 的存在。故而在官方基础上 fork 并增加了 windows 的支持（[已提交 PR](https://github.com/elastic/ansible-beats/pull/17)，但是官方不一定给合并 [捂脸] ）。关于 Ansible 管理 windows 可以参考我之前写的一篇文章  [Ansible2.7 批量管理 Windows](https://juejin.im/post/5c644d7ef265da2dea050f66)。

<!-- more -->

## 实验环境

| 类型         | 系统                          | ip           |
| ------------ | ----------------------------- | ------------ |
| Server(主控) | Ubuntu Server 16.04.5 LTS X64 | 192.168.0.22 |
| Client(受控) | Windows Server 2008 R2 SP1    | 192.168.0.23 |
| Clinet(受控) | Ubuntu Server 16.04.5 LTS X64 | 192.168.0.24 |
| Clinet(受控) | CentOS 7.6.1810 (Core)        | 192.168.0.25 |

> 注意: 主控端需要安装 Ansible 2.7.12 可参考

## 步骤

此处已假设主控端已安装 Ansible 2.7+，被控端的 Windows 的 WinRM 已配置完成

### 安装[anjia0532.ansible_beats](https://galaxy.ansible.com/anjia0532/ansible_beats)

在主控端(192.168.0.22)执行以下命令

```bash
root@ubuntu:/root/# ansible-galaxy install anjia0532.ansible_beats
- downloading role 'ansible_beats', owned by anjia0532
- downloading role from https://github.com/anjia0532/ansible-beats/archive/master.tar.gz
- extracting anjia0532.ansible_beats to /root/.ansible/roles/anjia0532.ansible_beats
- anjia0532.ansible_beats (master) was installed successfully
```

### 创建 inventorys

创建 `inventorys/hosts.yml`

```yaml
beats:
  hosts:
    192.168.0.23:
      ansible_user: Administrator
      ansible_password: password
      ansible_connection: winrm
      ansible_winrm_transport: basic
      ansible_port: 5985
    192.168.0.24:
      ansible_user: root
      ansible_ssh_private_key_file: /root/.ssh/id_rsa
    192.168.0.25:
      ansible_user: root
      ansible_ssh_private_key_file: /root/.ssh/id_rsa
```

### 创建 task

创建  beats.yml

```yaml
- name: Example playbook for installing packetbeat
  hosts: beats
  roles:
    - {
        role: anjia0532.ansible_beats,
        beat: "packetbeat",
        beat_conf:
          {
            "interfaces": { "device": "any" },
            "protocols":
              {
                "dns": { "ports": [53], "include_authorities": true },
                "http": { "ports": [80, 8080, 8000, 5000, 8002] },
                "memcache": { "ports": [11211] },
                "mysql": { "ports": [3306] },
                "pgsql": { "ports": [5432] },
                "redis": { "ports": [6379] },
                "thrift": { "ports": [9090] },
                "mongodb": { "ports": [27017] },
              },
          },
        output_conf: { "elasticsearch": { "hosts": ["localhost:9200"] } },
      }
  vars:
    use_repository: true
```

### 安装 beats

```bash
# ansible-playbook -i inventorys/hosts.yml ./beats.yml

// 忽略输出
PLAY RECAP *******************************************************************************************************************************************************************************************************************************************************************
192.168.0.23               : ok=17   changed=17    unreachable=0    failed=0
192.168.0.24               : ok=19   changed=19    unreachable=0    failed=0
192.168.0.25               : ok=19   changed=19    unreachable=0    failed=0
```

表明都成功了

### 查看配置文件和日志

```bash
ssh root@192.168.0.24
cat /etc/packetbeat/packetbeat.yml
```

```yaml
################### packetbeat Configuration #########################

############################# packetbeat ######################################
interfaces:
  device: any
protocols:
  dns:
    include_authorities: true
    ports:
      - 53
  http:
    ports:
      - 80
      - 8080
      - 8000
      - 5000
      - 8002
  memcache:
    ports:
      - 11211
  mongodb:
    ports:
      - 27017
  mysql:
    ports:
      - 3306
  pgsql:
    ports:
      - 5432
  redis:
    ports:
      - 6379
  thrift:
    ports:
      - 9090

###############################################################################
############################# Libbeat Config ##################################
# Base config file used by all other beats for using libbeat features

############################# Output ##########################################

output:
  elasticsearch:
    hosts:
      - localhost:9200

############################# Logging #########################################

logging:
  files:
    rotateeverybytes: 10485760
```

```bash
# less /var/log/packetbeat/packetbeat

2019-04-13T09:38:44.865+0800    INFO    instance/beat.go:611    Home path: [/usr/share/packetbeat] Config path: [/etc/packetbeat] Data path: [/var/lib/packetbeat] Logs path: [/var/log/packetbeat]
2019-04-13T09:38:44.868+0800    INFO    instance/beat.go:618    Beat UUID: 8fbd86a8-0bbc-4349-8aca-d4dc8c897ba2
2019-04-13T09:38:44.868+0800    INFO    [seccomp]       seccomp/seccomp.go:116  Syscall filter successfully installed
2019-04-13T09:38:44.868+0800    INFO    [beat]  instance/beat.go:931    Beat info       {"system_info": {"beat": {"path": {"config": "/etc/packetbeat", "data": "/var/lib/packetbeat", "home": "/usr/share/packetbeat", "logs": "/var/log/packetbeat"}, "type": "packetbeat", "uuid": "8fbd86a8-0bbc-4349-8aca-d4dc8c897ba2"}}}
2019-04-13T09:38:44.868+0800    INFO    [beat]  instance/beat.go:940    Build info      {"system_info": {"build": {"commit": "1d55b4bd9dbf106a4ad4bc34fe9ee425d922363b", "libbeat": "6.7.1", "time": "2019-04-02T15:15:12.000Z", "version": "6.7.1"}}}
2019-04-13T09:38:44.868+0800    INFO    [beat]  instance/beat.go:943    Go runtime info {"system_info": {"go": {"os":"linux","arch":"amd64","max_procs":4,"version":"go1.10.8"}}}
2019-04-13T09:38:44.872+0800    INFO    [beat]  instance/beat.go:947    Host info       {"system_info": {"host": {"architecture":"x86_64","boot_time":"2019-04-12T20:58:45+08:00","containerized":true,"name":"localhost.localdomain","ip":["127.0.0.1/8","::1/128","172.60.20.116/24","fe80::536d:17d0:e9f6:57c/64"],"kernel_version":"3.10.0-957.el7.x86_64","mac":["00:50:56:9f:8b:b7"],"os":{"family":"redhat","platform":"centos","name":"CentOS Linux","version":"7 (Core)","major":7,"minor":6,"patch":1810,"codename":"Core"},"timezone":"CST","timezone_offset_sec":28800,"id":"cd7bb2d0c80a41c89bb5b596c22fc85e"}}}
2019-04-13T09:38:44.873+0800    INFO    [beat]  instance/beat.go:976    Process info    {"system_info": {"process": {"capabilities": {"inheritable":null,"permitted":["chown","dac_override","dac_read_search","fowner","fsetid","kill","setgid","setuid","setpcap","linux_immutable","net_bind_service","net_broadcast","net_admin","net_raw","ipc_lock","ipc_owner","sys_module","sys_rawio","sys_chroot","sys_ptrace","sys_pacct","sys_admin","sys_boot","sys_nice","sys_resource","sys_time","sys_tty_config","mknod","lease","audit_write","audit_control","setfcap","mac_override","mac_admin","syslog","wake_alarm","block_suspend"],"effective":["chown","dac_override","dac_read_search","fowner","fsetid","kill","setgid","setuid","setpcap","linux_immutable","net_bind_service","net_broadcast","net_admin","net_raw","ipc_lock","ipc_owner","sys_module","sys_rawio","sys_chroot","sys_ptrace","sys_pacct","sys_admin","sys_boot","sys_nice","sys_resource","sys_time","sys_tty_config","mknod","lease","audit_write","audit_control","setfcap","mac_override","mac_admin","syslog","wake_alarm","block_suspend"],"bounding":["chown","dac_override","dac_read_search","fowner","fsetid","kill","setgid","setuid","setpcap","linux_immutable","net_bind_service","net_broadcast","net_admin","net_raw","ipc_lock","ipc_owner","sys_module","sys_rawio","sys_chroot","sys_ptrace","sys_pacct","sys_admin","sys_boot","sys_nice","sys_resource","sys_time","sys_tty_config","mknod","lease","audit_write","audit_control","setfcap","mac_override","mac_admin","syslog","wake_alarm","block_suspend"],"ambient":null}, "cwd": "/", "exe": "/usr/share/packetbeat/bin/packetbeat", "name": "packetbeat", "pid": 9683, "ppid": 1, "seccomp": {"mode":"filter"}, "start_time": "2019-04-13T09:38:44.350+0800"}}}
2019-04-13T09:38:44.874+0800    INFO    instance/beat.go:280    Setup Beat: packetbeat; Version: 6.7.1
2019-04-13T09:38:44.874+0800    INFO    elasticsearch/client.go:164     Elasticsearch url: http://localhost:9200
2019-04-13T09:38:44.875+0800    INFO    [publisher]     pipeline/module.go:110  Beat name: localhost.localdomain
2019-04-13T09:38:44.875+0800    INFO    procs/procs.go:101      Process watcher disabled
2019-04-13T09:38:44.877+0800    INFO    instance/beat.go:402    packetbeat start running.
2019-04-13T09:38:44.877+0800    INFO    [monitoring]    log/log.go:117  Starting metrics logging every 30s
2019-04-13T09:39:14.887+0800    INFO    [monitoring]    log/log.go:144  Non-zero metrics in the last 30s        {"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":160,"time":{"ms":162}},"total":{"ticks":610,"time":{"ms":614},"value":0},"user":{"ticks":450,"time":{"ms":452}}},"handles":{"limit":{"hard":4096,"soft":1024},"open":7},"info":{"ephemeral_id":"691a2203-1433-44d4-b173-938a52dbea22","uptime":{"ms":30042}},"memstats":{"gc_next":36201104,"memory_alloc":18724416,"memory_total":23093208,"rss":45715456}},"libbeat":{"config":{"module":{"running":0}},"output":{"type":"elasticsearch"},"pipeline":{"clients":0,"events":{"active":0}}},"system":{"cpu":{"cores":4},"load":{"1":0.1,"15":0.06,"5":0.05,"norm":{"1":0.025,"15":0.015,"5":0.0125}}}}}}
```

## 参考资料

- [Latest Releases via Apt (Ubuntu)](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#latest-releases-via-apt-ubuntu)
- [Windows Guides](https://docs.ansible.com/ansible/latest/user_guide/windows.html)
- [Document npcap requires WinPcap Compatible Mode](https://github.com/elastic/beats/issues/4364)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
