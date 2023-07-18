---
title: 080-RKE2自动更新K8S证书
urlname: rke2-renew-certs
date: '2023-01-09 19:35:21 +0800'
tags:
  - k8s
  - 容器
  - 云原生
  - kubernetes
categories:
  - kubernetes
  - 云原生
---

> 这是坚持技术写作计划（含翻译）的第 80 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解 Rancher RKE2 自动更新 K8S 证书。

<!-- more -->

## 背景介绍

[RKE2](https://docs.rke2.io/) 和 [RKE](https://rancher.com/docs/rke/latest/en/) ,[K3S](https://docs.k3s.io/) 一样，是 Rancher 系列的 K8S 发行版之一。

根据官方文档 [高级选项和配置#证书轮转](https://docs.rke2.io/advanced#certificate-rotation) 介绍，RKE2 证书默认是 12 个月有效期，过期之前 90 天内，重启 service(rke2-server 或者 rke2-agent) 将自动轮转证书。 在 v1.21.8+rke2r1 及以上版本，支持手动显示轮转证书。

不重启 service 或者早于过期之前 90 天重启，都不会触发证书轮转。

新手很容易忽略这个问题，等遇到时，往往已经发生故障了。记忆深刻那种。

## 如何操作

```bash
# 查询单个节点k8s证书有效期
date --date="$(openssl s_client -showcerts -connect localhost:6443   < /dev/null | openssl x509 -enddate -noout|cut -d= -f 2)" --iso-8601

# 查询单个节点k8s证书还有多少天过期
echo $(( ($(date --date="$(openssl s_client -showcerts -connect localhost:6443   < /dev/null | openssl x509 -enddate -noout|cut -d= -f 2)" +%s) - $(date +%s) )/(60*60*24) ))

# 查看 rke2-server 节点证书啥时候过期
for pem in /var/lib/rancher/rke2/server/tls/*.crt; do
   printf '%s: %s\n' \
      "$(date --date="$(openssl x509 -enddate -noout -in "$pem"|cut -d= -f 2)" --iso-8601)" \
      "$pem"
done | sort

# 查看 rke2-agent 节点证书啥时候过期
for pem in /var/lib/rancher/rke2/agent/*.crt; do
   printf '%s: %s\n' \
      "$(date --date="$(openssl x509 -enddate -noout -in "$pem"|cut -d= -f 2)" --iso-8601)" \
      "$pem"
done | sort
```

为了简化操作，编写了 bash 脚本（注意目前仅在 Ubuntu 20.04 LTS 测试通过，其他发行版请自行测试）

```bash
#!/usr/bin/env bash

# author anjia0532@gmail.com
# blog https://anjia0532.github.io/
# github https://github.com/anjia0532

RED='\033[0;31m'
NC='\033[0m' # No Color

usage () {
    echo -e "${RED} Renew certs of k8s cluster
    Usage : $0 {OPTIONS}
        -d < days between 1 and 90 e.g. 70 > ${NC}";
}

# parse args
while getopts "d:" opts; do
   case ${opts} in
      d) DAYS=${OPTARG} ;;
      *) usage; exit;;
   esac
done

# those args must be not null
if [ ! "$DAYS" ]
then
    usage
    exit 1
fi
# in batches to restart service
RANDOM_DAYS=$((1 + $RANDOM % 10))
echo -e "Max days is ${RED}${DAYS}${NC}, will plus ${RED}${RANDOM_DAYS} days${NC}"

DAYS=$((DAYS+RANDOM_DAYS))
echo -e "Max days is ${RED}${DAYS}${NC}"


if  [ "$DAYS" -gt 90 ]
then
echo -e "Max days > 90, Must be set it to 90"
DAYS=90
fi

# days between 1 and 90
if  [ "$DAYS" -gt 90 ] || [ "$DAYS" -lt 1 ]
then
    usage
    exit 1
fi

EXPIRE_DAYS=$(( ($(date --date="$(openssl s_client -showcerts -connect localhost:6443   < /dev/null | openssl x509 -enddate -noout|cut -d= -f 2)" +%s) - $(date +%s) )/(60*60*24) ))
SERVICE_NAME=$(systemctl --type=service --state=running | grep rke2 | awk '{print $1}')

if  [ $EXPIRE_DAYS -gt ${DAYS} ]
then
  echo -e "K8S certs will expire after ${RED} ${EXPIRE_DAYS} days > ${DAYS} days , skip renew ${NC}"
else
  echo -e "K8S certs will expire after ${RED} ${EXPIRE_DAYS} days <= ${DAYS} days , will be restart ${SERVICE_NAME} to renew certs ${NC}"
fi

[[ $EXPIRE_DAYS -le ${DAYS} ]] && ( echo -e "\n sudo systemctl restart ${SERVICE_NAME}" && sudo systemctl restart ${SERVICE_NAME})
```

将上面脚本保存成 k8s_renew_certs.sh

```bash
chmod +x k8s_renew_certs.sh
./k8s_renew_certs.sh -h
 Renew certs of k8s cluster
    Usage : /etc/rancher/k8s_renew_certs.sh {OPTIONS}
        -d < days between 1 and 90 e.g. 70 >
```

`-d`设置过期天数，大于 1 小于该值则会重启，比如如果 20 天内过期则重启，该值为 20， rke2 在 90 天内，重启 service 会自动轮转证书，所以该值需要 `1 < days < 90`，大于 90 的会强制改成 90

为了防止定时任务统一时间全部重启，可能导致服务不稳定，加了随机种子, `-d+random[1,10]`,
比如
-d 90, 则强制为 90
-d 88, 如果随机数是 1，则实际是 89，如果随机数大于等于 2，强制为 90
-d 70, 则 70+[1,10] 最终范围为 [71,80]
-d 0 ,报错退出

脚本会自动判断是 rke2-server 还是 rke2-agent

## rke2-canal 报错

`failed to setup network for sandbox "xxxxx": error getting ClusterInformation: connection is unauthorized: Unauthorized`

```bash
kubectl rollout restart ds rke2-canal -n kube-system
```

参考 [RKE2 ContainerNetwork interface ( Not able to create any running pods anymore) #3425](https://github.com/rancher/rke2/issues/3425)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- 我的博客
- 我的掘金
