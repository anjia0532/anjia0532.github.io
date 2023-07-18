---
title: 058-差不多是Java中最好用的Emoji库了
urlname: emoji-java
date: '2021-03-03 19:35:21 +0800'
tags:
  - java
  - emoji
categories:
  - java
---

> 这是坚持技术写作计划（含翻译）的第 58 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解 java 中最好用的 emoji 库 [vdurmont/emoji-java](https://github.com/vdurmont/emoji-java) ,以及我提交的 PR[ #175](https://github.com/vdurmont/emoji-java/pull/175) 用于解决 emoji-java 表情更新不及时，及本地化方面的改进

<!-- more -->

## 什么是 emoji

emoji 是表情符号/颜文字/绘文字，与表情包堪称现代人社交离不开的两个功能，区别是，表情包一般是用于非正式场合甚至带点恶搞的意思，而 emoji 稍微正式些（比如拍领导马屁专用，赞:👍，拍手:👏）,
而发给领导![](https://cdn.nlark.com/yuque/0/2021/jpeg/226273/1614756451123-08486131-0e0b-4673-8281-0a994e0f8fc2.jpeg#align=left&display=inline&height=240&originHeight=240&originWidth=437&size=0&status=done&style=none&width=437)，则会有失业风险

在一些国际网络交流中，emoji 能够比较方便的增加双方交流，比如不存在的幽灵网站，某推，某书，某 ins，emoji 非常流行

欢迎大家来吐槽下，你是啥时候更深入的了解到 emoji 的(日常聊天除外)?或者说，有多少人是因为 mysql 阉割版 utf8 而被 emoji 坑过？

一般用户昵称，用户评论属于重灾区。。。

解决方案无非两个，
1，将 emoji 替换成空字符串，以 java 为例`str.replaceAll("^\\p{L}\\p{M}\\p{N}\\p{P}\\p{Z}\\p{Cf}\\p{Cs}\\p{Sc}\\s]", "")`
2，修改 mysql 列类型为 utf8mb4 或者换数据库 pgsql 等

## emoji-java 简介及使用

如果有人用过[hutool 工具包](https://www.hutool.cn/docs/#)，那可能有用过如下语法 参考 [Emoji 工具-EmojiUtil](https://www.hutool.cn/docs/#/extra/emoji/Emoji%E5%B7%A5%E5%85%B7-EmojiUtil)

```java
String alias = EmojiUtil.toAlias("😄");//:smile:
String emoji = EmojiUtil.toUnicode(":smile:");//😄
String alias = EmojiUtil.toHtml("😄");//&#128102;
```

此处，hutool 就是基于 [vdurmont/emoji-java](https://github.com/vdurmont/emoji-java) 库来做的

简单来说，emoji-java 库是一个 java 的 emoji 工具库，主要用于判断文本是否含有 emoji 表情，将 emoji 替换为文本描述(比如将 😄 替换成 `:smile:` ),也可以将描述，逆向成 emoji 表情，比如 `:smile:` 😄

移除 emoji 表情

```java
String str = "An 😀awesome 😃string with a few 😉emojis!";
Collection<Emoji> collection = new ArrayList<Emoji>();
collection.add(EmojiManager.getForAlias("wink")); // This is 😉

System.out.println(EmojiParser.removeAllEmojis(str));
System.out.println(EmojiParser.removeAllEmojisExcept(str, collection));
System.out.println(EmojiParser.removeEmojis(str, collection));

// Prints:
// "An awesome string with a few emojis!"
// "An awesome string with a few 😉emojis!"
// "An 😀awesome 😃string with a few emojis!"
```

基本使用，参见他的[README.md](https://github.com/vdurmont/emoji-java/blob/master/README.md),功能很简单，也很实用
目前支持的所有 emoji，参见他的[EMOJIS.md](https://github.com/vdurmont/emoji-java/blob/master/EMOJIS.md) ，里面没有列出来的，则暂未支持

是的，他采取的是白名单的做法，列出的是 emoji，没有列出的，则不是 emoji

下面例子可能会更方便理解

```java
// https://unicode.org/emoji/charts/full-emoji-list.html#1f636_200d_1f32b_fe0f
EmojiManager.isEmoji("😶‍🌫️") //false
```

具体案例表情参见 [https://unicode.org/emoji/charts/full-emoji-list.html#1f636_200d_1f32b_fe0f](https://unicode.org/emoji/charts/full-emoji-list.html#1f636_200d_1f32b_fe0f)
与 emoji 13.1 规范相比， emoji-java 5.1.1 一共缺失 244 个表情，详见 我提的 [issues](https://github.com/vdurmont/emoji-java/issues/169)

对于一个公共库来说，算是缺失比较多的了

## 对 emoji-java 打补丁

针对上一个问题，我抽空提交了一个[feat(module): add emoji-json-generator module #175](https://github.com/vdurmont/emoji-java/pull/175)
以后可以基于[https://unicode.org/emoji/charts/full-emoji-list.html](https://unicode.org/emoji/charts/full-emoji-list.html) 自己生成 emoji.json，能够保证实时性

```bash
mvn exec:java -Dexec.mainClass="com.vdurmont.emoji.JsonGenerator" [-Dexec.args="proxy=1270.0.1 port=1080 \
    path=path\to\already\download\emoji.html save_url=/path/to/emoji.json \
    url=https://unicode.org/emoji/charts/full-emoji-list.html \
    emoji_path=/path/to/already/emoji.json \
    emoji_i18n_path=/path/to/i18n_description/emoji_i18n.json
    "]
```

简单说明下

- []：包裹的，都是选填的，不写没关系
- proxy 和 port: 是防止访问[https://unicode.org/emoji/charts/full-emoji-list.html](https://unicode.org/emoji/charts/full-emoji-list.html) 失败，可以自己设置代理服务器（比如国外或者香港的）
- path：如果不想每次实时访问 [https://unicode.org/emoji/charts/full-emoji-list.html](https://unicode.org/emoji/charts/full-emoji-list.html) 可以预先下载到本地，然后读本地的，这是路径地址
- save_url: 这是重新生成的 emoji.json 地址
- url: 防止 [https://unicode.org/emoji/charts/full-emoji-list.html](https://unicode.org/emoji/charts/full-emoji-list.html) 改地址
- emoji_path: 是为了兼容 emoji-java 官方库 [https://github.com/anjia0532/emoji-java/blob/master/src/main/resources/emojis.json](https://github.com/anjia0532/emoji-java/blob/master/src/main/resources/emojis.json) ，防止从 [https://unicode.org/emoji/charts/full-emoji-list.html](https://unicode.org/emoji/charts/full-emoji-list.html) 生成的跟之前的不一样，导致库里对应不上，如果设置了 emoji_path,并且能够匹配上的，以 emoji_path 的为准
- emoji_i18n_path:如果是想将 👀 对应的解释从 `eyes`   换成中文 `两只眼睛` ，那么就设置需要翻译的 json 路径，格式参考 [https://github.com/anjia0532/emoji-java/blob/master/emoji-table-generator/src/main/resources/emojis.i18n.json](https://github.com/anjia0532/emoji-java/blob/master/emoji-table-generator/src/main/resources/emojis.i18n.json)

自己生成的 emoji.json 如何使用呢
第一种方案：
自己 fork 并生成 emoji.json 覆盖原来的 emoji.json 并打包发到私服上，这样原代码不用动，兼容性最好，改动最小

第二种方案:
自己 fork 并修改 EmojiManager，因为他本身写死了，不支持外部传入 emoji.json 的路径或者文件流

第三种方案:
在自己项目里参考 EmojiManager 自行实现类似的静态工具类 主要是用 EmojiLoader 加载 emoji.json 转成 List<Emoji>，并转换成 EmojiTrie

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/03/03/emoji-java)
- [我的掘金](https://juejin.cn/post/6935373449686679589/)
