---
title: jdk绿色免安装
date: 2017-05-17 12:39:16
tags: java
---

## windows 免安装

java自从被oracle收购后，windows下新的版本只有安装版。没有zip免安装。

windows安装版有一下坏处
1. 会写注册表
2. 会将java.exe,javaw.exe 等解压到`C:\Windows\System32`或者`C:\Windows\SysWOW64` 
3. 会将定期更新程序设置开机自启动，发现新版本弹窗提示
4. 会在`PATH`中写一个oracle的javapath,还会加上`jre\bin`

好处就是安装方便


今天给同事处理问题时，就因为他电脑装了jdk7和jdk8两个安装版，并且path配置的`%JAVA_HOME%\bin;`又配了一个`%JAVA_HOME%\jre\bin;`导致出了一个很诡异的错误。


下面说一下，如何免安装

从 http://www.oracle.com/technetwork/java/javase/downloads/index.html 下载最新的jdk windows安装版
e.g.
`jdk-8u131-windows-x64.exe`

用解压缩软件解压到`E:\jdk-8u131-windows-x64\` `Win+R`->`cmd`打开命令行

```
cd /d E:\jdk-8u131-windows-x64\.rsrc\1033\JAVA_CAB10
extrac32.exe 111

:: 此时解压出 tools.zip 文件
:: 打开当前文件夹
explorer.exe .
:: 将tools.zip 用解压软件解压到当前文件夹,e.g. `E:\jdk-8u131-windows-x64\.rsrc\1033\JAVA_CAB10\tools`

:: 将 .pack文件改成.jar文件

cd tools
for /r %x in (*.pack) do .\bin\unpack200 -r "%x" "%~dx%~px%~nx.jar"

:: 解压 src.zip 如果不需要源码 src.zip 可忽略此步

cd ..\..\JAVA_CAB9
extrac32 110

:: 将src.zip移动到tools文件夹

move src.zip ..\JAVA_CAB10\tools\

:: 将tools文件夹里的内容复制到指定目录，e.g. D:\jdk

xcopy /s /e /i /y E:\jdk-8u131-windows-x64\.rsrc\1033\JAVA_CAB10\tools d:\jdk

:: 删除 E:\jdk-8u131-windows-x64 文件夹
cd / && rd /s /q E:\jdk-8u131-windows-x64
```

设置环境变量 
增加 `JAVA_HOME` `d:\jdk`

修改`PATH`

追加 `;%JAVA_HOME%\bin;`

增加 `CLASSPATH`

`.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;`

设置环境变量后，需要重新打开`cmd`

```
java -version && javac -version
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b15, mixed mode)
javac 1.8.0_131
```


## linux 免安装
```bash

# 下载文件
$ wget -P ~/downloads --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u121-b13/e9e7ea248e2c4826b92b3f075a80e441/jdk-8u121-linux-x64.tar.gz

# 解压
$ sudo tar zxf ~/downloads/jdk-*.tar.gz -C /usr/local/

#创建软连接
$ sudo ln -sf /usr/local/jdk1.8.0_121 /usr/local/jdk

$ sudo vi /etc/profile

#设置java环境
export JAVA_HOME=/usr/local/jdk
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar;:$JAVA_HOME/lib/tools.jar:$CLASSPATH
export PATH=$JAVA_HOME/bin:$PATH

#保存并退出

#使配置生效
$ source /etc/profile
```

本人原创

博客 https://anjia.ml/2017/05/17/jdk-zip/
简书 http://www.jianshu.com/p/5dc20d5d4f5c
掘金 https://juejin.im/post/591bdb222f301e006bcde36b