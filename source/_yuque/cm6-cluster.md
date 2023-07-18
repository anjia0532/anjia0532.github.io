---
title: 020-CM配置集群
urlname: cm6-cluster
date: '2019-04-29 18:10:00 +0800'
tags:
  - CDH
  - hadoop
  - 大数据
categories: 大数据
---

> 这是坚持技术写作计划（含翻译）的第 20 篇，定个小目标 999，每周最少 2 篇。

本文主要介绍，如何配置 CM 集群。

<!-- more -->

### 提示

- 安装完后，打开  [http://masterIP:7180/](http://masterip:7180/)  进行登录，用户名/密码 是 admin/admin

- 安装过程中，如果出现

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557474684611-ff6a7f1c-4d35-4041-b8c7-67748cafcb4e.png#align=left&display=inline&height=845&originHeight=845&originWidth=1456&size=26862&status=done&width=1456)和![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558168684154-4141c4e8-4592-40a3-8c07-11ec9d3e8ed9.png#align=left&display=inline&height=612&originHeight=612&originWidth=897&size=25849&status=done&width=897)
多按几次 Ctrl+Shift+R 即可

### 选择协议

注意授权问题，建议选择 Cloudera Express 版，防止使用试用版收费项目后，60 天过期后停用掉导致的问题。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557473083139-d59ce28b-5999-429d-9d51-db825bf714b7.png#align=left&display=inline&height=928&originHeight=928&originWidth=1764&size=122298&status=done&width=1764)

### 配置集群

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557473101214-998d7c3b-e202-4fe7-800f-b08da5dea3f1.png#align=left&display=inline&height=900&originHeight=900&originWidth=1434&size=69964&status=done&width=1434)
填上 FQDN 或者 ip，多台用英文逗号隔开，然后点搜索。并选中需要安装的服务器，点击继续。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557473272389-edb6fff0-2c54-4ffd-9495-ce76f709c196.png#align=left&display=inline&height=915&originHeight=915&originWidth=1450&size=109471&status=done&width=1450)
国内使用官方公开库，那等装完了，起码得以天为单位，此处使用自建 repo，参考  [018-CDH6.2 构建本地源加速 CDH 安装](https://juejin.im/post/5cc57a01f265da036c57929b) , 如果嫌麻烦，并且中科大的免费镜像能用的话，可以用中科大的，参考  [ustclug/mirrorrequest#56](https://github.com/ustclug/mirrorrequest/issues/56)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557473386274-c92e6d15-465f-46ed-9989-d1e2aede4972.png#align=left&display=inline&height=886&originHeight=886&originWidth=1446&size=164237&status=done&width=1446)
选择更多选项，把乱七八糟的都删掉，只留一个即可。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557473404683-1cdb865e-3ee0-4f07-bc3c-97138f76869d.png#align=left&display=inline&height=461&originHeight=461&originWidth=1701&size=46026&status=done&width=1701)

如果已经安装了 jdk 1.8 可以跳过。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557473446912-ef5d1472-ec49-4cc3-a4b5-ab803b54aa20.png#align=left&display=inline&height=908&originHeight=908&originWidth=1444&size=288573&status=done&width=1444)
填主机密码或者 ssh key
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557473472053-b318c2b8-6611-456b-9962-85e0fbcf2c17.png#align=left&display=inline&height=899&originHeight=899&originWidth=1421&size=101355&status=done&width=1421)
点击继续，即会在对应主机执行安装操作。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557473917009-483cee28-4940-4aab-be3a-d2ec921d2eeb.png#align=left&display=inline&height=889&originHeight=889&originWidth=1207&size=52000&status=done&width=1207)
点继续，进行健康检查，如果是 Inspect Network 和 Hosts 都是绿色的，则继续，否则可以点击，显示检查结果，进行排查，排查，改正后，点击重新运行。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557474626008-4d55b976-68d8-4af0-b02e-c9f8644ce89c.png#align=left&display=inline&height=914&originHeight=914&originWidth=1451&size=106906&status=done&width=1451)
选择需要安装的模块，点击继续
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557474843163-6ad18e46-c3f1-4daa-8810-68d956649901.png#align=left&display=inline&height=840&originHeight=840&originWidth=1431&size=141716&status=done&width=1431)
配置角色（各个主机需要安装的组件）
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557474977110-d9ba7095-a8eb-4c9d-b273-9750fb0596da.png#align=left&display=inline&height=397&originHeight=397&originWidth=1701&size=41273&status=done&width=1701)
配置元数据库信息，点击测试连接（如果提示找不到数据库驱动，需要参考之前的文章 [017-Centos7.6+CDH 6.2 安装和使用](https://juejin.im/post/5cd4c949f265da03a158463a)，安装驱动）
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557475069567-67acf641-1d6a-4d7d-a3a6-45a7845eb337.png#align=left&display=inline&height=805&originHeight=805&originWidth=1443&size=138020&status=done&width=1443)
点击继续后，会显示安装进度。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557475628886-18547c11-a4b2-4f26-9132-29ad74f282aa.png#align=left&display=inline&height=906&originHeight=906&originWidth=1440&size=139785&status=done&width=1440)
点击继续，则安装结束。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557475636171-04c27f36-723a-4eff-b461-f9407e2faef4.png#align=left&display=inline&height=322&originHeight=322&originWidth=1213&size=12755&status=done&width=1213)

### 参考资料

- [018-CDH6.2 构建本地源加速 CDH 安装](https://juejin.im/post/5cc57a01f265da036c57929b)
- [ustclug/mirrorrequest#56](https://github.com/ustclug/mirrorrequest/issues/56)
- [017-Centos7.6+CDH 6.2 安装和使用](https://juejin.im/post/5cd4c949f265da03a158463a)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
