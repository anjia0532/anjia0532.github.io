---
title: 051-ffmpeg 研究笔记
urlname: ffmpeg-note
date: '2021-01-12 20:21:35 +0800'
tags:
  - python
  - ffmpeg
  - 音视频
categories:
  - ffmpeg
---

> 这是坚持技术写作计划（含翻译）的第 51 篇，定个小目标 999，每周最少 2 篇。

工作原因，需要对视频进行处理，趁机会对 ffmpeg 进行学习，并随手记录

<!-- more -->

## 关于 [ffmpeg ](https://ffmpeg.org/)和[libav](https://libav.org/) 的区别

ffmpeg 基本成了开源音视频处理的基座了，能接触到的大部分商用，个人开发的音视频软件都有 ffmpeg，比如格式工厂，EV 录屏，QQ 影音，暴风影音等等 基本上你用 everything 等软件全局搜一下 ffmpeg 就知道了
libav 是 ffmpeg 成员闹掰后 fork 出的一个分支，两者算是同源并且相互补充，经常互相合并
基本上 ffmpeg 更底层一些，libav 更上层些

此处说的 ffmpeg 和 libav 是指的项目名（广义上的）
以两个视频合并为例

```bash
# ffmpeg
ffmpeg -i "concat:1.mp4|2.mp4" -codec copy full.mp4

# libav
avconv -i "concat:1.mp4|2.mp4|3.mp4" -codec copy full.mp4
```

## 常用库

基本主流编程语言都对 ffmpeg 进行了封装，如果是轻量使用的话，会比直接自己背 ffmpeg 的命令会友好一些

- [Python-ffmpeg-python](https://github.com/kkroening/ffmpeg-python)
- [Java-ffmpeg-cli-wrapper](https://github.com/bramp/ffmpeg-cli-wrapper)
- [golang-goav](https://github.com/giorgisio/goav)

## 常用语法

- 合并视频(拼接)

```bash
ffmpeg -i "concat:1.mp4|2.mp4" -codec copy full.mp4
```

[参考 stackoverflow 上的 ffmpeg 三种拼接视频的方法](https://stackoverflow.com/a/11175851/7001350)

- 九宫格/分屏视频拼接

```bash
ffmpeg  -re  -i  test.mp4 -re -i test.mp4 -re  -i  test.mp4 -re  -i  test.mp4 -filter_complex "nullsrc=size=1920x1080 [base];[0:v] setpts=PTS-STARTPTS,scale=960x540 [upperleft]; [1:v] setpts=PTS-STARTPTS, scale=960x540 [upperright]; [2:v] setpts=PTS-STARTPTS, scale=960x540 [lowerleft]; [3:v] setpts=PTS-STARTPTS, scale=960x540 [lowerright];[base][upperleft] overlay=shortest=1[tmp1]; [tmp1][upperright] overlay=shortest=1:x=960 [tmp2]; [tmp2][lowerleft] overlay=shortest=1:y=540 [tmp3];[tmp3][lowerright] overlay=shortest=1:x=960:y=540" -c:v libx264 out_1080p.mp4
```

解释下

```bash
-re -i 多文件输入

-filter_complex 滤镜

nullsrc=size=1920x1080 [base] 创建基础画布，尺寸是宽1920,高1080,颜色默认是绿色，如果要自定义颜色，需要用color filter，例如 color=s=1920x1080:c=black （黑色背景）
[0:v] setpts=PTS-STARTPTS,scale=960x540 [upperleft];
    其中 [0:v] 是0输入的视频部分，同理，[1:v] 是第二个视频,[0:a]是第一个视频的音频部分
    setpts=PTS-STARTPTS 帧是按照时间戳顺序从每个输入视频中获取的，因此最好将所有叠加输入通过setpts = PTS-STARTPTS过滤器传递，以使它们以相同的零时间戳开始，从0开始计数PTS 就是用PTS-STARTPTS，如果要加速setpts=0.5*PTS，慢速setpts=2.0*PTS
    scale=960x540是缩放，可以强制尺寸，也可以设置同比缩放，scale = 320:-1,设置宽是320，高-1意味着是根据320同比缩放高度，scale = iw*0.5:-1 宽度缩放一半,高度同比，如果是指向缩放，不想放大 scale='min(iw,320)':-1, 如果宽度小于320像素，不缩放，大于320使用320，并且长度同比缩放[upperleft]是一路流(stream)
    如果一组stream有多个filter，使用英文逗号:,分隔，别用;分隔
[base][upperleft] overlay=shortest=1[tmp1] [base]是底层画布[upperleft]是左上方视频（名字随便起） overlay=shortest=1 overlay 是说[upperleft]悬浮到[base]左上方，（因为默认不写坐标是x=0,y=0,所以是左上方）,shortest=1 是说，两个视频合成的总视频长度，以最短的为终点(比如，1 30秒，2 40秒，最终的视频就是30秒), 如果想指定xy位置，可以参考 overlay=x=960:y=540，指的是悬浮视频处于背景画布的x=960:y=540

-c:v libx264 指的是视频编码器用 libx264
```

另外一个例子

```bash
ffmpeg  -i 1.mp4 -i 2.mp4 -v libx264 -ac 2 -filter_complex "[0:v]scale=320:240[a];[a]pad=640:240[b];[b][1:v]overlay=320:0[out]" -map "[out]"  -map 1:a  c.mp4
```

```java
[a]pad=640:240[b]: 是保持a视频画面不变的情况下(320:240)，将视频尺寸放大到640:240,空白部分用黑屏填充
[b][1:v]overlay=320:0[out] 将第二个视频[1:v]悬浮到[b]视频上，x=320,y=0,其实就是将原本的两个视频，强制压缩成320:240,并且左右分布拼接成一个
-map "[out]" 用[out]做视频
-map 1:a 用第二个视频的声音做音频
```

参考 [ffmpeg 实现多宫格效果，视频拼接合成](https://www.cnblogs.com/famhuai/p/10276081.html) ，参考 [ffmpeg:多路视频合并之九宫格特效:福优学苑](https://www.bilibili.com/video/BV1254y1W7RK)

- 添加水印/文字/字幕/画中画

如果是要添加视频/图片类型的，可以使用上面介绍的 overlay 即可

```bash
ffmpeg -i 1.mp4 -i 1.png -filter_complex  overlay=0:0 1.mp4
```

```bash
ffmpeg -i 1.mp4 -vf "drawtext=fontfile=/usr/share/fonts/TTF/Vera.ttf:text='%{pts\:hms}': x=(w-tw)/2: y=h-(2*lh): fontcolor=white" 2.mp4
```

```bash
-vf 视频滤镜,-af 是音频滤镜，filter_complex 和 lavfi是一样的
drawtext 在视频上添加文本
fontfile=/usr/share/fonts/TTF/Vera.ttf 文本文件，如果只是字母或者数字，可以不写，用默认字体就行了
text='%{pts\:hms}' 这是duration，也可以不用变量，自己写
x=(w-tw)/2:y=h-(2*lh) 设置存在的x,y值
fontcolor 文本颜色
```

参考 [ffmpeg # drawtext 进阶](https://www.jianshu.com/p/d9d8ee30d621/) ，[《ffmpeg basics》中文版 -- 10.在视频上添加文本](https://blog.csdn.net/qq_34305316/article/details/103937279?depth_1-utm_source=distribute.pc_relevant.none-task-blog-OPENSEARCH-4&utm_source=distribute.pc_relevant.none-task-blog-OPENSEARCH-4)

- 插入空白帧/空白视频/音频

```bash
ffmpeg -y -loglevel warning -hide_banner -stats -vsync 2 -i 2.mp4 -filter_complex "[0:a]adelay=4s:all=true[a0];[0:v]scale=368:640,tpad=start_duration=4[v0];[v0][a0]concat=n=1:v=1:a=1[v][a]" -map "[v]" -map "[a]" 3.mp4
```

```bash
-y 如果同名直接覆盖，不需要交互
-loglevel warning 只打印warning级别日志
-hide_banner 隐藏 banner
-stats 显示进度
-vsync 2 输入帧从解码器到编码器，时间戳保持不变；如果出现相同时间戳的帧，则丢弃之
[0:a]adelay=4s:all=true[a0] 第一个音频延迟4秒
[0:v]scale=368:640,tpad=start_duration=4[v0] 第一个视频前面填充4秒空白视频
[a0]concat=n=1:v=1:a=1[v][a] 拼接起来
```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/01/12/ffmpeg-note/)
- [我的掘金](https://juejin.im/post/6916755968416022535)
