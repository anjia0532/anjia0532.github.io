---
title: 076-快速开发HelmCharts（工具篇）
urlname: build-helm-charts
date: '2022-08-18 19:35:21 +0800'
tags:
  - k8s
  - helm
categories:
  - k8s
---

> 这是坚持技术写作计划（含翻译）的第 76 篇，定个小目标 999，每周最少 2 篇。

本文不是用来讲解 helm 干嘛的，他的基础概念一类的。这种的网上一搜一大把。本文主要是讲解如何利用工具插件来提升开发 Helm Chart 效率，降低出错概率。

<!-- more -->

## 官方最佳实践

不使用工具的情况下，最快上手的看到是官方文档 [最佳实践](https://helm.sh/zh/docs/chart_best_practices/conventions/)

通过 `helm create 应用名(比如 doris-charts)`创建一个骨架
装了 [Kubernates](https://plugins.jetbrains.com/plugin/10485-kubernetes) 插件的话，也可以从 Idea 创建
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660874799097-588bae27-e0c1-453e-a482-0ffdad3b2b2a.png#clientId=u2bdd351d-7275-4&from=paste&height=460&id=ua1a2382e&originHeight=460&originWidth=564&originalType=binary∶=1&rotation=0&showTitle=false&size=86859&status=done&style=none&taskId=uf9adb867-f304-4f5f-9fa8-5624d194a51&title=&width=564)

## 快速跑通

使用类似 Rancher kubesphere 等带 UI 的系统，快速验证服务可用。
带 UI 的比一次次手动改 yaml 舒服多了。
最好是单独起一个命名空间。

```yaml
for n in $(kubectl get -n default -o=name pvc,configmap,serviceaccount,secret,ingress,service,deployment,statefulset,hpa,job,cronjob)
do
mkdir -p $(dirname $n)
kubectl get -o=yaml $n > $n.yaml
done
```

把该命名空间的配置文件导出。然后人工复制粘贴到 通过 helm create 生成的骨架模板里。

## 快速开发

此处以 [Idea](https://www.jetbrains.com/idea/)+[Kubernates](https://plugins.jetbrains.com/plugin/10485-kubernetes)插件([官方文档](https://www.jetbrains.com/help/idea/kubernetes.html))为例

![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660819846652-784c2bbf-a26d-4afb-add5-363dea270597.png#clientId=u894cd9a1-0132-4&from=paste&height=711&id=ud9b59c0f&originHeight=711&originWidth=1235&originalType=binary∶=1&rotation=0&showTitle=false&size=629463&status=done&style=none&taskId=uc2ac7b9f-822f-4e2d-a4d8-7fabb5af4eb&title=&width=1235)
当然你也可以用命令 `helm lint PATH [flags]` 文档 [Helm Lint 验证](https://helm.sh/zh/docs/helm/helm_lint/)

这就省了你一趟趟的 helm uninstall && helm install 或者 helm template 试错了。

会把一些 values 里的数据直接替换到 charts 模板文件里 写的时候比较方便。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660821148556-f1e32101-3ced-47b5-9212-1f0e3cec8b0c.png#clientId=u894cd9a1-0132-4&from=paste&height=543&id=ub74b788b&originHeight=543&originWidth=823&originalType=binary∶=1&rotation=0&showTitle=false&size=430686&status=done&style=none&taskId=u91e3998b-2229-4065-8584-e934e59d204&title=&width=823)
也可以把单个模板渲染出来（类似 helm template）
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660874882926-e9e074e8-d293-4901-a58c-c68e719e86c6.png#clientId=u2bdd351d-7275-4&from=paste&height=523&id=u9a9d5c5a&originHeight=523&originWidth=1836&originalType=binary∶=1&rotation=0&showTitle=false&size=618328&status=done&style=none&taskId=ua77ee804-4df4-4887-ab81-3137620b74c&title=&width=1836)

其他还有很多功能，比如查看应用日志，执行 Shell 命令，转发端口到本地，修改应用的 yaml 并 apply

## 快速部署

以 [Idea](https://www.jetbrains.com/idea/)+[Alibaba Cloud Toolkit](https://plugins.jetbrains.com/plugin/11386-alibaba-cloud-toolkit) 为例

```bash
cd /tmp/doris-charts && sudo helm  template doris --create-namespace -n doris-system /tmp/doris-charts -f /tmp/doris-charts/values.yaml | sudo kubectl apply -n doris-system -f -
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660820465900-63f55d28-89a3-4d4d-86fe-6261e8d3b513.png#clientId=u894cd9a1-0132-4&from=paste&height=1023&id=uce4f1490&originHeight=1023&originWidth=1049&originalType=binary∶=1&rotation=0&showTitle=false&size=185983&status=done&style=none&taskId=u81a5d51e-c8e3-49ea-9a60-ead91cd8d78&title=&width=1049)
这样每次只需要点击运行按钮就行（之所以用 helm template | kubectl apply 就是省了一趟趟 helm uninstall && helm install）。
如果本身用传统扔 Jar/War 的方式部署，或者依赖阿里云产品比较重的话，用这个工具还不错。
比如
一键发布(打包，传输到主机上，kill 原进程，启动新进程，并且 tail 前 N 条日志)，Shell 到远程主机终端（省了 Xshell 了）
一键构建镜像，启动 DockerCompose
发布到阿里云微服务相关产品

但是说实在的，这个插件平时我是禁用的，随用随开，他有这么几个问题

1. 往往不支持最新版，每次升级后，大概率需要等一周左右才会发布新版
2. 会让电脑变卡，集成了太多功能了，比如 Arthas，比如 PMD 静态分析，尤其是 PMD，特别卡
3. 每次启动 Idea 这个插件都得加载一大会（虽然可以最小化，但是仍然很不爽）
4. 会往杭州一个 OSS 传输一些日志，咱也不知道干嘛，反正关不掉，我直接改本地 HOST，变相禁掉了。

![lQLPJxaZMK77LpvNAR_NAnCwRSUmLYFcMpEC_A2W_EAzAA_624_287.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1660820939973-894dcd11-8b81-4a24-8537-1ae15b28df4d.png#clientId=u894cd9a1-0132-4&from=paste&height=287&id=uf31e6486&originHeight=287&originWidth=624&originalType=binary∶=1&rotation=0&showTitle=false&size=23918&status=done&style=none&taskId=u236134d0-d19f-4e3e-91be-7852d288c22&title=&width=624)

## 另外

还有一个神器 [nocalhost](https://plugins.jetbrains.com/plugin/16058-nocalhost) ,缺点是不支持最新的 2022.2.x ，可以自行编译。
功能挺多，参考官网 [https://nocalhost.dev/zh-CN/docs/quick-start/](https://nocalhost.dev/zh-CN/docs/quick-start/)。

## 总结

1. Helm Charts 作为一个交付产物，比发给别人一堆 yaml 好管理多了(有版本，可以自定义值)。
2. Kubernates 插件跟 NocalHost 插件有不少功能是重合的，但是各自有各自的特点，建议按需使用。
3. Alibaba Cloud Toolkit 过于臃肿，但是某些场景下确实也能省不少事，见仁见智吧。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/08/18/build-helm-charts/)
- [我的掘金](https://juejin.cn/post/7133412011815206926/)
