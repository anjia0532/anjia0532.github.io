---
title: '082-天文摄影软件(PPIP,AS3,RS6)汉化教程'
urlname: localization
date: '2023-08-01 19:35:21 +0800'
tags:
  - 汉化
categories:
  - 汉化
---

> 这是坚持技术写作计划（含翻译）的第 82 篇，定个小目标 999。

本文有点偏，最近入坑摄影，拍月亮接触了一些天文摄影软件，但是英文的，就想着汉化下，看着顺眼（当然网上有别人汉化好的）

<!-- more -->

对于 PPIP 和 AS3,RS6 不熟悉的，可以参考 [富人靠装备，穷人靠后期: 如何用 400 块的镜头给月亮拍大头贴](https://zhuanlan.zhihu.com/p/339893346)

本文不讨论三个软件的用法，仅仅讲解如何自己汉化这三个软件（网上一堆别人汉化过的，如果不介意，可以直接百度自行下载）

PS [sequator](https://sites.google.com/view/sequator/download)（需科学上网） 用此方法汉化失败（可能是加壳了）

## Radialix 3 或者 Sisulizer 4

因为 PPIP，AS，RS 本身都没有做防护，汉化起来比较简单，这俩软件都可以，看自己喜好了

下载 [Radialix 3](https://pan.baidu.com/share/init?surl=pJID8gV&pwd=erzr) 参考 [汉化入门教程（绝对适合新手）](https://www.52pojie.cn/thread-371237-1-1.html)

下载 [Sisulizer 4](https://pan.baidu.com/s/1nmWhHg9aWPDAmayXncbTRA?pwd=6atp) 授权码 `LAENT628901447012574111` 参考 [【新手向】三分钟学会软件汉化，谁都看得懂](https://www.bilibili.com/video/BV16p4y1C7jT/?vd_source=dee839563659f580b03eb813381554f3)

## 汉化 PPIP

下载 [ppip-2.5.9](https://web.archive.org/web/20230531163522/https://sites.google.com/site/astropipp/downloads)（需科学上网）并安装。

下载 .NET 反编译调试软件 [dnSpy](https://github.com/dnSpy/dnSpy/releases/tag/v6.1.8) 用法参考 [神器如 dnSpy，无需源码也能修改 .NET 程序](https://blog.csdn.net/WPwalter/article/details/80457131)

Ctrl+O 打开 PPIP.exe 文件

Ctrl+Shift+K 搜索关键字 （注意改成数字/字符串选项，勾选区分大小写），比如 About

双击搜索结果，Ctrl+Shift+E 编辑方法

把 `this.aboutToolStripMenuItem.Text = "About";` 改成 `this.aboutToolStripMenuItem.Text = "关于";` 点击右下角 `编译`

![](./Snipaste_2023-08-04_14-26-37.png#id=okpZD&originalType=binary∶=1&rotation=0&showTitle=false&status=done&style=none&title=)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/226273/1691143841280-e57609ea-d290-40e3-ae83-2a6164115cb6.png#averageHue=%23468677&clientId=ua346bbe2-1466-4&from=paste&height=925&id=u9c3514da&originHeight=925&originWidth=1895&originalType=binary∶=1&rotation=0&showTitle=false&size=213107&status=done&style=none&taskId=u2c490fef-26ed-4a09-925e-f7d8cc6a912&title=&width=1895)
![](./Snipaste_2023-08-04_14-29-58.png#id=DwPhv&originalType=binary∶=1&rotation=0&showTitle=false&status=done&style=none&title=)![image.png](https://cdn.nlark.com/yuque/0/2023/png/226273/1691143864767-f7bd1e13-30d6-4e43-a118-97809820ee39.png#averageHue=%2386a9a1&clientId=u13dbe056-be77-4&from=paste&height=417&id=u55de258a&originHeight=417&originWidth=1200&originalType=binary∶=1&rotation=0&showTitle=false&size=89626&status=done&style=none&taskId=u937f1d14-d61a-4a05-b816-df717f4ca91&title=&width=1200)

剩下的就是体力活了

## 汉化 AutoStakkert3(AKA AS3)

下载 [AutoStakkert_3.1.4_x64.zip](https://www.astrokraai.nl/software/AutoStakkert_3.1.4_x64.zip) 并解压

运行 Radialix 3 或者 Sisulizer 4 ，本文以 Sisulizer 4 为例

![](./Snipaste_2023-08-02_12-04-42.png#id=utmPJ&originalType=binary∶=1&rotation=0&showTitle=false&status=done&style=none&title=)![image.png](https://cdn.nlark.com/yuque/0/2023/png/226273/1691143893823-4a258924-c906-423e-9897-42eb22da10c1.png#averageHue=%23ececeb&clientId=u13dbe056-be77-4&from=paste&height=604&id=ufb41a79e&originHeight=604&originWidth=1489&originalType=binary∶=1&rotation=0&showTitle=false&size=94096&status=done&style=none&taskId=u14601ab1-71d6-4cef-a4f9-9ad4cb187bf&title=&width=1489)

基本上一路下一步就行，创建后，记得保存工程

![](./Snipaste_2023-08-04_15-10-32.png#id=sjOVy&originalType=binary∶=1&rotation=0&showTitle=false&status=done&style=none&title=)![image.png](https://cdn.nlark.com/yuque/0/2023/png/226273/1691143904038-39a8a7e9-5590-4cd0-b525-399e2f9e8b2b.png#averageHue=%23e1c581&clientId=u13dbe056-be77-4&from=paste&height=951&id=uffa468dc&originHeight=951&originWidth=1430&originalType=binary∶=1&rotation=0&showTitle=false&size=262268&status=done&style=none&taskId=u2f3c9c4c-d9bc-45d5-964a-ff14ea8b621&title=&width=1430)

注意第一步是右键选中，选择属性，第四步，原来默认是 `<sl>/<file>` 也就是在根目录创建 `zh/AutoStakkert.exe` 这样会找不到 dll 文件，改成 `<sl><file>` 也就是 `zh/AutoStakkert.exe`

剩下的就是体力活了+1

## 汉化 RegiStax 6 (AKA RS6)

下载并安装 [RegiStax 6](http://www.astronomie.be/registax/updateregistax6.exe)

依然以 Sisulizer 4 为例

忽略创建工程步骤，基本上还是一路下一步

![](./Snipaste_2023-08-02_12-04-42-3.png#id=eutOg&originalType=binary∶=1&rotation=0&showTitle=false&status=done&style=none&title=)![image.png](https://cdn.nlark.com/yuque/0/2023/png/226273/1691143917249-18a5ff04-8399-4f53-989b-0d4569c1cfb0.png#averageHue=%23f0edec&clientId=u13dbe056-be77-4&from=paste&height=961&id=u86c3c51d&originHeight=961&originWidth=1696&originalType=binary∶=1&rotation=0&showTitle=false&size=191474&status=done&style=none&taskId=u9af0a5da-087c-4669-b23e-128860ab90a&title=&width=1696)

注意跟汉化 AS3 一样 ，设置属性，改成 `<sl><file>`

剩下的就是体力活了+2

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2023/08/04/localization)
- 我的掘金
