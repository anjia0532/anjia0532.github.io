---
title: 【译】单应用跨多k8s集群的部署与管理
urlname: rancher2-k8s-multi-cluster-app
date: 2019-02-15 14:08:00 +0800
tags: [k8s,rancher,rancher2,集群,运维,翻译]
categories: [k,8,s]
---

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1550147773034-220bf087-600f-47dd-9d24-65ac37f79d65.png#align=left&display=inline&height=371&name=image.png&originHeight=371&originWidth=746&size=38965&width=746)

## 引言

近日(春节前后)，Rancher labs 宣布，其旗下的开源企业级 Kubernetes 管理平台 Rancher，发布了 Rancher 2.2 Preview 2（预览版 2），它包含了许多在 k8s  集群操作的强大特性

可以通过访问[发布页面](https://rancher.com/products/rancher/2.2/)和[发布说明](https://github.com/rancher/rancher/releases/tag/v2.2.0-alpha6)来了解所发布的新功能

本文将介绍其中一个特性：多集群应用(multi-cluster applications)，下面将为您介绍，该特性将如何显著减少您的工作量，并提高多集群操作的可靠性。

## 概览

假如您有用过 k8s，并且有两个及以上的集群运维经验，那么您遇到下面的情况

- 当跨多个可用区部署(AZs)时，应用需要具有更高的容错性
- 在具有数百个集群的边缘计算场景中，同一个应用需要在多个集群上运行。

在高可靠性的情况下，运维操作人员通常通过将节点从多个可用区纳入到一个集群内来降低单个可用区不可用风险。但是这个方案的问题在于，虽然抵抗了可用区故障，但是防不住集群本身故障，集群故障的可能性高于可用区故障，而且一旦集群出故障后，可能会影响集群中在运行的程序。

另外一种方法是，每个可用区中运行单独的集群，病症每个集群上运行应用程序的副本。相当于每个可用区都有一套 k8s 集群，但是每个集群手动维护应用程序成本高，又易错。

边缘计算场景跟多可用区集群相同的问题：应用程序手动维护，既耗时，又容易出错，即使运维团队给力，创建了复杂的脚本来部署和升级，但是又多了一个故障点，而且这些脚本也需要升级和维护，并且要求负责的运维人员不仅要编写流程（升级发版流程），还要在脚本失败时能够转成人肉运维。

从[Rancher 2.2 Preview 2](https://github.com/rancher/rancher/releases/tag/v2.2.0-alpha6) 开始，Rancher 支持在任意数量的 k8s 集群中同时部署和升级同一应用程序的副本。

同时也扩展了基于 Helm 软件包管理器的应用商店(Application Catalogs)，在此之前，应用商店仅适用于单个集群,我们在全局级别增加了一个附加功能，权限允许的情况下，可以将应用程序部署到 Rancher 管理的任意集群上。

有关 Rancher 2.2 Preview2 的功能的完整演示，请加入 [Rancher2 月份的在线 MeetUp](https://rancher.com/events/2019/meetup-multi-cluster-apps/) ，届时将提供新功能的演示，并在 Q&A 环节进行答疑。

下面将演示，在 Rancher 中如何便捷的管理多集群应用。

## 功能快速入门

- 登陆 Rancher 后，将看到纳管的所有集群的列表，同时在菜单栏新增了一个 `多集群应用(Multi-Cluster Apps)`  的按钮

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210384952-8bbbfcb0-e3d7-42f2-b55f-456323970275.png#align=left&display=inline&height=265&originHeight=568&originWidth=1600&size=0&width=746)

- 单击 `多集群应用`  按钮后，将看到两个按钮，`管理Catalogs`  和 `启动` 。`管理Catalogs`  将跳到 `应用商店(Catalogs)`  的管理页，您开源在其中启用主要 Helm repo 或者添加其他第三方 Helm repo。
- 单击 `启动`  按钮以启动新应用程序。

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210400369-8dd2c530-bffc-484b-8081-310fb69f06b7.png#align=left&display=inline&height=319&originHeight=685&originWidth=1600&size=0&width=746)

- 从显示的可以部署的应用中，选择 Grafana(用于演示)。

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210419223-c687bf46-93e1-428d-82bc-754a460cdc4b.png#align=left&display=inline&height=298&originHeight=640&originWidth=1600&size=0&width=746)

- 按照要求配置详细信息，使用表单或者直接用提供 YAML 进行配置，注意，在此处的设置将应用到部署此应用程序的集群中。

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210430368-f065ea71-821c-4458-bd14-7457822473bb.png#align=left&display=inline&height=520&originHeight=1115&originWidth=1600&size=0&width=746)

- 在 `配置选项`  下，在  `Target（目标）`  下拉框中选择目标集群的指定项目。

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210444391-421fb9d3-d2a5-45ed-89e7-5e4accca5793.png#align=left&display=inline&height=392&originHeight=840&originWidth=1600&size=0&width=746)

- 选择升级策略。此处为了演示，我们将选择 `滚动更新`  并提供每批 1 个，间隔 20 秒。此设置可以确保以后升级应用时，一次只更新一个集群，并且每个集群升级操作的间隔为 20 秒。

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210458439-2fd561f6-6ea1-4fdb-bb69-83edb7de2f01.png#align=left&display=inline&height=370&originHeight=794&originWidth=1600&size=0&width=746)

- 如果要调整集群间的差异，可以在 `Answer Overrides`  部分进行设置。

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210474235-75246694-14bc-410b-badc-f4bbd81c492d.png#align=left&display=inline&height=270&originHeight=579&originWidth=1600&size=0&width=746)

- 一切准备妥当，点击底部 `启动` ,然后将跳到结果页，显示刚刚已安装的多集群应用(此处是演示用的 Grafana)。每个应用将显示当前状态和目标集群以及项目列表。

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210490267-a0d665be-27f1-45f6-9988-a2e81c93a3c8.png#align=left&display=inline&height=140&originHeight=301&originWidth=1600&size=0&width=746)

- 当应用程序可以升级时，应用状态将显示  `Upgrade Available`
- 要启动升级，请单击应用上的菜单按钮（三个点的菜单），然后选择升级

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1550210508332-34b12a85-bcb1-4706-9795-2f839c8aa88d.png#align=left&display=inline&height=287&originHeight=615&originWidth=1600&size=0&width=746)

- 验证是否已选择 `滚动更新`  选项
- 更改一些设置，然后点击底部的 `升级`  按钮

打开目标集群的 `工作负载`  选项卡，将看到其中一个状态更改为 `更新` ,此集群中的应用将被更新，然后 Rancher 将暂停 20s（刚刚设置的间隔时间）,然后继续更新下一个集群的应用。

## 总结

多集群应用程序将减少运维团队的工作量，并使跨集群快速可靠的部署和升级应用成为可能。
要在实验室或者开发环境中测试这些功能，请安装[最新的 Alpha 版本](https://rancher.com/docs/rancher/v2.x/en/installation/server-tags/#helm-chart-repositories)，如果有任何反馈意见，请在[Github 上提交 Issues 或](https://github.com/rancher/rancher/issues)者加入[论坛](https://forums.rancher.com/) 或[Slack](https://slack.rancher.io/) 。

## 参考资料

- [rancher 博客原文--Introducing Multi-Cluster Applications in Rancher 2.2 Preview 2](https://rancher.com/blog/2019/introducing-multi-cluster-apps/)
- [Rancher 中国微信公众号--革命性新特性 | 单一应用跨多 Kubernetes 集群的部署与管理](https://mp.weixin.qq.com/s/yfE22D04D98r8e7BAlD3qg)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
