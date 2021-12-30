---
title: 060-轻量级基于curl的seafile上传脚本
urlname: seafile-client-curl
date: 2021-04-07 12:35:21 +0800
tags: [bash,linux,seafile]
categories: [linux,bash]
---

> 这是坚持技术写作计划（含翻译）的第 60 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解利用 bash 脚本在 linux 通过命令行上传文件到 seafile 服务器

<!-- more -->

seafile 有基于 python 2.x 的 client，但是年代比较久远，另外为了在服务器上转存文件就装个 python2.x 有点大材小用了，看了下 seafile 的 api 文档，比较简单，就基于 curl 撸了个 bash 脚本。

目前只有简单的上传单文件功能。

## 使用

### 如何使用

将下面脚本保存成 `seafile-upload.sh` ,并 `chmod+x seafile-upload.sh`

```bash
#!/usr/bin/env bash

# this script depend on jq,check it first
RED='\033[0;31m'
NC='\033[0m' # No Color
if ! command -v jq &> /dev/null
then
    echo -e "${RED}jq could not be found${NC}, installed and restart plz!\n"
    exit
fi


usage () { echo "Usage : $0 -u <username> -p <password> -h <seafile server host> -f <upload file path> -d <parent dir default value is /> -r <repo id> -t <print debug info switch off/on,default off>"; }

# parse args
while getopts "u:p:h:f:d:r:t:" opts; do
   case ${opts} in
      u) USER=${OPTARG} ;;
      p) PASSWORD=${OPTARG} ;;
      h) HOST=${OPTARG} ;;
      f) FILE=${OPTARG} ;;
      d) PARENT_DIR=${OPTARG} ;;
      r) REPO=${OPTARG} ;;
      t) DEBUG=${OPTARG} ;;
      *) usage; exit;;
   esac
done

# those args must be not null
if [ ! "$USER" ] || [ ! "$PASSWORD" ] || [ ! "$HOST" ] || [ ! "$FILE" ] || [ ! "$REPO" ]
then
    usage
    exit 1
fi

# optional args,set default value

[ -z "$DEBUG" ] && DEBUG=off

[ -z "$PARENT_DIR" ] && PARENT_DIR=/

# print vars key and value when DEBUG eq on
[[ "on" == "$DEBUG" ]] && echo -e "USER:${USER} PASSWORD:${PASSWORD} HOST:${HOST} FILE:${FILE} PARENT_DIR:${PARENT_DIR} REPO:${REPO} DEBUG:${DEBUG}"

# login and get token
TOKEN=$(curl -s --location --request POST "${HOST}/api2/auth-token/" --header 'Content-Type: application/x-www-form-urlencoded' --data-urlencode "username=${USER}" --data-urlencode "password=${PASSWORD}" | jq -r ".token")

[ -z "$TOKEN" ] && echo -e "${RED}login seafile faild${NC}, call your administrator plz!\n" && exit 1

# gen upload link
UPLOAD_LINK=$(curl -s --header "Authorization: Token ${TOKEN}" "${HOST}/api2/repos/${REPO}/upload-link/?p=${PARENT_DIR}" | jq -r ".")

[ -z "$UPLOAD_LINK" ] && echo -e "${RED}get upload link faild${NC}, call your administrator plz!\n" && exit 1

# upload file
UPLOAD_RESULT=$(curl -s --header "Authorization: Token ${TOKEN}" -F file="@${FILE}" -F filename=$(basename ${FILE}) -F parent_dir="${PARENT_DIR}" -F replace=1 "${UPLOAD_LINK}?ret-json=1")

[ -z "$UPLOAD_RESULT" ] && echo -e "${RED}faild to upload ${FILE}${NC}, call your administrator plz!\n" && exit 1

# print upload result
[[ "on" == "$DEBUG" ]] && echo -e "TOKEN:${TOKEN} UPLOAD_LINK:${UPLOAD_LINK} UPLOAD_RESULT:${UPLOAD_RESULT}"
```

```bash
# ubuntu
apt install -y jq
# centos
# yum install -y jq
chmod +x ./seafile-upload.sh

./seafile-upload.sh -u <username> -p <password> -h <seafile server host> -f <upload file path> -d <parent dir default value is /,must be start with /> -r <repo id> -t <print debug info switch off/on,default off>
```

### 参数说明

- -u seafile 用户名
- -p seafile 密码
- -h seafile 地址，https://xxx.xxx.com
- -f 需要上传的文件路径，比如/tmp/test.zip
- -d 上传目录，必须是`/`开头
- -r 资料库 id
- -t 是否开启 debug 模式，如果是`on` 则会打印参数，否则啥都不打印

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/04/07/seafile-client-curl/)
- [我的掘金](https://juejin.cn/post/6948276424633483300/)
- [官方 api 文档 Quick Start](https://download.seafile.com/published/web-api/v2.1/file-upload.md)
- [官方 python-seafile client 文档](https://github.com/haiwen/python-seafile/blob/master/doc.md)
