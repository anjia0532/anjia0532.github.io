title: hexo搭建博客
date: 2017-02-03 16:07:16
tags: [blog,hexo]
categories: [programming]
---
本文主要讲解如何通过github pages功能从零开始搭建一个炫酷的个人技术博客

<!-- more -->

## 配置环境
### Nodejs
[安装Nodejs](http://www.runoob.com/nodejs/nodejs-install-setup.html)

默认安装在c盘,具体的默认参数可以通过 `npm config ls -l` 进行查看,输出类似下面的信息, 注意 `; ...` 开头的都是注释内容,不生效

```bash

; cli configs
long = true
scope = ""
user-agent = "npm/4.0.5 node/v7.4.0 win32 x64"

; builtin config undefined
; prefix = "C:\\Users\\{userName}\\AppData\\Roaming\\npm" (overridden)


; cache = "C:\\Users\\{userName}\\AppData\\Roaming\\npm-cache" (overridden)
cache-lock-retries = 10
cache-lock-stale = 60000
cache-lock-wait = 10000
cache-max = null

```

修改默认库路径

```bash

npm config set cache "${NodejsHome}\node_cache" # 将${NodejsHome}换成实际安装路径

npm config set prefix "${NodejsHome}"

```

`npm config set prefix` 设置成安装路径的好处是 `npm install -g xxx` 安装的库在执行时不会报命令找不到(否则还需要改系统的`Path`环境变量)

天朝网络环境比较差,需要使用 [淘宝npm镜像](http://npm.taobao.org/)

```bash

npm install -g cnpm --registry=https://registry.npm.taobao.org

```

安装成功后,以后使用`npm install`的统统可以改成`cnpm install`



### Git

下载地址: [http://git-scm.com/download/](http://git-scm.com/download/)

### Hexo

```bash

cd d:\blog # 创建目录

cnpm install hexo-cli -g # 全局安装hexo

hexo init # 初始化当前目录(hexo init blog 创建blog并初始化)

cnpm install # 使用淘宝npm镜像加载依赖

hexo g # 生成静态代码

hexo s # 启动服务,在http://localhost:4000/查看

```

打开 [http://localhost:4000/](http://localhost:4000/) 已经可以看到默认的一篇blog了

```bash

# 命令缩写

hexo n == hexo new

hexo g == hexo generate

hexo s == hexo server

hexo d == hexo deploy

# 命令组合
hexo d -g # 生成并部署

hexo s -g # 生成并本地预览

```

如果是windows打开git-bash.exe

### GitHub 配置

#### 生成rsa文件

```bash

ssh-keygen

# 输入编译代码
Enter file in which to save the key (/c/Users/{userName}/.ssh/id_rsa): # rsakey文件名,假设使用默认的id_rsa
Enter passphrase (empty for no passphrase): # 密码
Enter same passphrase again: #确认密码

```

#### 文本编辑器打开 ~/.ssh/id_rsa.pub 并复制内容

```
ssh-rsa

xxxx 具体的key xxxxxx  userName@email

```

#### github 设置ssh key

左上角 用户->settings->Personal settings->SSH and GPG keys->New SSH key->Title 随意->Key 贴上一步的ssh-rsa开头的一串文本 ->Add SSH key

#### 创建仓库

左上角 用户旁边 + 号->New repository->Repository name 填${userName}.github.io ${userName}为账号名->Create repository

### 提交github并自动发布

#### 提交代码到github

```bash

git init # 初始化本地仓库

git add . # 添加文件

git commit -m '初始化' # 提交到本地仓库并指定message

git checkout -b blog-source # 创建分支,为了使用 travis-ci 自动构建

git remote add origin git@github.com:${userName}/${userName}.github.io.git # 添加远程仓库地址 将 ${userName} 替换成实际账户名

git push origin blog-source:blog-source # 推送到github远程仓库分支

```

#### 创建 `.travis.yml` 文件

```yml
language: node_js
node_js: stable

# S: Build Lifecycle
install:
  - npm install


before_script:
  - npm install hexo-helper-qrcode --save
  - npm install hexo-generator-feed --save

script:
  - hexo g

after_script:
  - cd ./public
  # 如果设置自定义域名,则自动生成CNAME文件
  - if [ $MY_DOMAIN ]; then echo ${MY_DOMAIN} > CNAME; fi 
  - git init
  - git config user.name "${userName}"
  - git config user.email "${email}"
  - git add .
  - git commit -m "Update docs"
  - git push --force --quiet "https://${GH_TOKEN}@${GH_REF}" master:master

branches:
  only:
    - blog-source
env:
 global:
   - GH_REF: github.com/${userName}/${userName}.github.io.git
   # 自定义域名
   - MY_DOMAIN: anjia.ml
```

将${userName}和${email}替换成实际值

参考 [简书-手把手教你使用Travis CI自动部署你的Hexo博客到Github上](http://www.jianshu.com/p/e22c13d85659)

通过travis自动构建并发布

### 使用https 自定义域名
[开启 Github Pages 自定义域名 HTTPS 和 HTTP/2 支持​](https://zhuanlan.zhihu.com/p/22667528)

