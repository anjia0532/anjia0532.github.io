---
title: 063-apisix基于acme.sh自动更新HTTPS证书
urlname: renew-hook-update-apisix
date: '2021-05-24 20:05:21 +0800'
tags:
  - nginx
  - openresty
  - apisix
  - kong
  - acme.sh
categories:
  - apisix
---

> 这是坚持技术写作计划（含翻译）的第 63 篇，定个小目标 999，每周最少 2 篇。

apisix 官方不支持自动更新 ssl 证书，但是开放了 api，本文主要讲解如何使用 acme.sh 的 renew-hook 特性来实现自动更新 apisix 的 https 证书。

<!-- more -->

## 创建/更新 apisix ssl 脚本

**NOTE:**
需要安装 openssl , jq , acme.sh

> 我的脚本，支持检索已有 ssl 列表是否存在该证书，如果存在则更新，不存在则创建，snis 使用 OpenSSL 读的 pem，有效起止时间也是用 OpenSSL 读的 pem

将[我的 gist](https://gist.github.com/anjia0532/9ebf8011322f43e3f5037bc2af3aeaa6#file-renew-hook-update-apisix-sh) 保存成可执行脚本,并添加执行权限

```bash
$ curl --output /root/.acme.sh/renew-hook-update-apisix.sh --silent https://gist.githubusercontent.com/anjia0532/9ebf8011322f43e3f5037bc2af3aeaa6/raw/65b359a4eed0ae990f9188c2afa22bacd8471652/renew-hook-update-apisix.sh

$ chmod +x /root/.acme.sh/renew-hook-update-apisix.sh

$ /root/.acme.sh/renew-hook-update-apisix.sh

Usage : /root/.acme.sh/renew-hook-update-apisix.sh -h <apisix admin host> -p <certificate pem file> -k <certificate private key file> -a <admin api key> -t <print debug info switch off/on,default off>
```

考虑到国内网络环境，所以此处贴出 renew-hook-update-apisix.sh 的代码，但是注意，后续更新以[gist](https://gist.github.com/anjia0532/9ebf8011322f43e3f5037bc2af3aeaa6#file-renew-hook-update-apisix-sh)为主

```bash
#!/usr/bin/env bash

# author anjia0532@gmail.com
# blog https://anjia0532.github.io/
# github https://github.com/anjia0532

# this script depend on jq,check it first
RED='\033[0;31m'
NC='\033[0m' # No Color

if ! [ -x "$(command -v jq)" ]; then
  echo  -e "${RED}Error: jq is not installed.${NC}" >&2
  exit 1
fi

if ! [ -x "$(command -v openssl)" ]; then
  echo  -e "${RED}Error: openssl is not installed.${NC}" >&2
  exit 1
fi

if ! [ -x "$(command -v ~/.acme.sh/acme.sh)" ]; then
  echo  -e "${RED}Error: acme.sh is not installed.(doc https://github.com/acmesh-official/acme.sh/wiki/How-to-install)${NC}" >&2
  exit 1
fi

usage () { echo "Usage : $0 -h <apisix admin host> -p <certificate pem file> -k <certificate private key file> -a <admin api key> -t <print debug info switch off/on,default off>"; }

# parse args
while getopts "h:p:k:a:t:" opts; do
   case ${opts} in
      h) HOST=${OPTARG} ;;
      p) PEM=${OPTARG} ;;
      k) KEY=${OPTARG} ;;
      a) API_KEY=${OPTARG} ;;
      t) DEBUG=${OPTARG} ;;
      *) usage; exit;;
   esac
done

# those args must be not null
if [ ! "$HOST" ] || [ ! "$PEM" ] || [ ! "$KEY" ] || [ ! "$API_KEY" ]
then
    usage
    exit 1
fi

# optional args,set default value

[ -z "$DEBUG" ] && DEBUG=off

# print vars key and value when DEBUG eq on
[[ "on" == "$DEBUG" ]] && echo -e "HOST:${HOST} API_KEY:${API_KEY} PEM FILE:${PEM} KEY FILE:${KEY} DEBUG:${DEBUG}"


# get all ssl and filter this one by sni name
cert_content=$(curl --silent --location --request GET "${HOST}/apisix/admin/ssl/" \
--header "X-API-KEY: ${API_KEY}" \
--header 'Content-Type: application/json' | jq "first(.node.nodes[]| select(.value.snis[] | contains(\"$(openssl x509 -in $PEM -noout -text|grep -oP '(?<=DNS:|IP Address:)[^,]+'|sort|head -n1)\")))")


validity_start=$(date --date="$(openssl x509 -startdate -noout -in $PEM|cut -d= -f 2)" +"%s")
validity_end=$(date --date="$(openssl x509 -enddate -noout -in $PEM|cut -d= -f 2)" +"%s")

# create a new ssl when it not exist
if [ -z "$cert_content" ]
then
  cert_content="{\"snis\":[],\"status\": 1}"

  # read domains from pem file by openssl
  snis=$(openssl x509 -in $PEM -noout -text|grep -oP '(?<=DNS:|IP Address:)[^,]+'|sort)
  for sni in ${snis[@]} ; do
    cert_content=$(echo $cert_content | jq ".snis += [\"$sni\"]")
  done

  cert_content=$(echo $cert_content | jq ".|.cert = \"$(cat $PEM)\"|.key = \"$(cat $KEY)\"|.validity_start=${validity_start}|.validity_end=${validity_end}")

  cert_update_result=$(curl --silent --location --request POST "${HOST}/apisix/admin/ssl/" \
  --header "X-API-KEY: ${API_KEY}" \
  --header 'Content-Type: application/json' \
  --data "$cert_content" )

  [[ "on" == "$DEBUG" ]] && echo -e "cert_content: \n${cert_content}\n\ncreate result json:\n\n${cert_update_result}"
else
  # get exist ssl id
  URI=$(echo $cert_content | jq -r ".key")
  ID=$(echo ${URI##*/})
  # get exist  ssl certificate json , modify cert and key value
  cert_content=$(echo $cert_content | jq ".value|.cert = \"$(cat $PEM)\"|.key = \"$(cat $KEY)\"|.id=\"${ID}\"|.update_time=$(date +'%s')|.validity_start=${validity_start}|.validity_end=${validity_end}")

  # update apisix ssl
  cert_update_result=$(curl --silent --location --request PUT "${HOST}/apisix/admin/ssl/${ID}" \
  --header "X-API-KEY: ${API_KEY}" \
  --header 'Content-Type: application/json' \
  --data "$cert_content" )

  [[ "on" == "$DEBUG" ]] && echo -e "cert_content: \n${cert_content}\n\nupdate result json:\n\n${cert_update_result}"
fi

exit 0
```

## acme.sh 相关

1. 安装 acme.sh

参考官方文档 [https://github.com/acmesh-official/acme.sh/wiki/How-to-install](https://github.com/acmesh-official/acme.sh/wiki/How-to-install)

```bash
curl https://get.acme.sh | sh -s email=my@example.com
```

2. 申请证书,并添加 renew-hook

参考官方文档 [https://github.com/acmesh-official/acme.sh/wiki/How-to-issue-a-cert](https://github.com/acmesh-official/acme.sh/wiki/How-to-issue-a-cert) 和 renew-hook 的文档 [https://github.com/acmesh-official/acme.sh/wiki/Using-pre-hook-post-hook-renew-hook-reloadcmd](https://github.com/acmesh-official/acme.sh/wiki/Using-pre-hook-post-hook-renew-hook-reloadcmd)

```bash
$ acme.sh  --issue  --staging  -d demo.domain --renew-hook "/root/.acme.sh/renew-hook-update-apisix.sh  -h http://apisix-admin:port -p /root/.acme.sh/demo.domain/demo.domain.cer -k /root/.acme.sh/demo.domain/demo.domain.key -a xxxxxxxxxxxxx"

$ acme.sh --renew --domain demo.domain
```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1621846151113-3d51d00f-c962-4b1c-a0c1-5478ba9d7e38.png#clientId=u1b96d101-bfc0-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=306&id=u32e1ee6b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=306&originWidth=1889&originalType=binary∶=1&rotation=0&showTitle=false&size=33444&status=done&style=none&taskId=uc13b01f7-d5c9-4bfa-9211-8df9a48bd67&title=&width=1889)

题外话，如果用 ansible 的话，也可以用我的[ansible galaxy](https://galaxy.ansible.com/anjia0532/acme_sh) ,文档参考 [https://github.com/anjia0532/ansible-acme-sh/blob/master/README.md](https://github.com/anjia0532/ansible-acme-sh/blob/master/README.md)

```bash
ansible-galaxy install anjia0532.acme_sh
```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/e78fa84f96fef4e733J60tq8EA~~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/05/24/renew-hook-update-apisix)
- [我的掘金](https://juejin.cn/post/6965778290619449351)
- [request help: How to auto upgrade SSL certificate, such as using let's encrypt #3841](https://github.com/apache/apisix/issues/3841)
