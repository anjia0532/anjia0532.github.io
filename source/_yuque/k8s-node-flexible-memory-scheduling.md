---
title: 069-k8s 集群基于节点内存水位线通过打污点方式来干预Pod调度
urlname: k8s-node-flexible-memory-scheduling
date: '2022-01-28 18:47:21 +0800'
tags:
  - k8s
  - kubernetes
categories:
  - k8s
---

> 这是坚持技术写作计划（含翻译）的第 69 篇，定个小目标 999，每周最少 2 篇。

本文主要介绍如何最小化改造的情况下，根据中小型 k8s 集群的 node 节点的内存，通过打污点(taints)的形式，柔性干预 Pod 调度

<!-- more -->

## 简单介绍

其实关于 k8s 调度方面的资料很多

比如 k8s 的 [Node Allocatable](https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/)，对我来说，有点硬了，可能导致短暂的高峰把 pod 给驱逐了

比如大佬 智博老师 提到的 [https://github.com/kubernetes-sigs/descheduler](https://github.com/kubernetes-sigs/descheduler) 和 [https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/trimaran/README.md](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/trimaran/README.md) 管用是管用，但是有点复杂了。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1643367626412-2c80474b-7edc-4196-874d-a904745991e7.png#clientId=u20a1d64f-3005-4&from=paste&height=614&id=uc084f177&originHeight=614&originWidth=1001&originalType=binary∶=1&rotation=0&showTitle=false&size=108337&status=done&style=none&taskId=uad603d68-f773-4101-b3f4-5466c99e2cc&title=&width=1001)

## 可用脚本

花了几分钟，写了个 bash 脚本，脚本放到了[我的 gist](https://gist.github.com/anjia0532/71e5a706ea01b79f98d8b1e27e9459a7) 上，防止过国内没法访问，下面也粘贴一份

```bash
#!/usr/bin/env bash

# author anjia0532@gmail.com
# blog https://anjia0532.github.io/
# github https://github.com/anjia0532

RED='\033[0;31m'
NC='\033[0m' # No Color

usage () {
    echo -e "${RED} Auto add/remove mem=poor:NoSchedule taint to your nodes of k8s cluster by threshold value(Low water level/High water level)
    Usage : $0 {OPTIONS}
        -k < kubectl command e.g. -k sudo kubectl --kubeconfig /path/to/config.yaml>
        -l < Low water level range (0,100) e.g. -l 50 >
        -h < high water level range (0,100) e.g. -h 80>
        -f < Selector (label query) to filter on, supports '=', '==', and '!='. e.g. -f key1=value1,key2=value2 > ${NC}";
}

# parse args
while getopts "k:l:h:f:" opts; do
   case ${opts} in
      k) KUBECTL_CMD=${OPTARG} ;;
      l) LOW=${OPTARG} ;;
      h) HIGH=${OPTARG} ;;
      f) FILTER=${OPTARG} ;;
      *) usage; exit;;
   esac
done

# those args must be not null
if [ ! "$KUBECTL_CMD" ] || [ ! "$LOW" ] || [ ! "$HIGH" ]
then
    usage
    exit 1
fi

# low and hign must between 0 and 100
if  [ "$LOW" -ge 100 ] || [ "$HIGH" -ge 100 ] || [ "$LOW" -le 0 ] || [ "$HIGH" -le 0 ]
then
    usage
    exit 1
fi

echo -e "View top node(order by memory) by this command \n
${RED} ${KUBECTL_CMD} top node --selector=\"${FILTER}\" --use-protocol-buffers --sort-by='memory' \n ${NC}"

nodes=()
while IFS='' read -r line; do nodes+=("$line"); done < <(${KUBECTL_CMD} top node --selector="${FILTER}" --no-headers --use-protocol-buffers --sort-by='memory' | awk '{print $1","$5}'| sed "s/%//g")

for node in "${nodes[@]}" ; do
    IFS="," read -r node_name mem_useage <<< "${node}"
    [[ $mem_useage -ge ${HIGH} ]] && ( echo -e "\n${KUBECTL_CMD} taint nodes ${node_name} mem=poor:NoSchedule --overwrite" && ${KUBECTL_CMD} taint nodes ${node_name} mem=poor:NoSchedule --overwrite)
    [[ $mem_useage -le ${LOW} ]] && ( echo -e "\n${KUBECTL_CMD} taint nodes ${node_name} mem=poor:NoSchedule-" && ${KUBECTL_CMD} taint nodes ${node_name} mem=poor:NoSchedule- )
done
```

核心部分就最后五六行，用法也很简单

```bash
k8s_node_flexible_memory_scheduling.sh
Auto add/remove mem=poor:NoSchedule taint to your nodes of k8s cluster by threshold value(Low water level/High water level)
 Usage : ./a.sh {OPTIONS}
 -k < kubectl command e.g. -k sudo kubectl --kubeconfig /path/to/config.yaml>
 -l < Low water level range (0,100) e.g. -l 50 >
 -h < high water level range (0,100) e.g. -h 80>
 -f < Selector (label query) to filter on, supports '=', '==', and '!='. e.g. -f key1=value1,key2=value2 >

k8s_node_flexible_memory_scheduling.sh -k "kubectl" -l 60 -h 80 -f "biz=demo"
node/192.168.1.139 modified
node/192.168.1.137 modified
node/192.168.1.126 modified
node/192.168.1.140 untainted
node/192.168.1.141 untainted
node/192.168.1.142 untainted
error: taint "mem:NoSchedule" not found
error: taint "mem:NoSchedule" not found

```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/e78fa84f96fef4e733J60tq8EA~~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/01/28/k8s-node-flexible-memory-scheduling/)
- [我的掘金](https://juejin.cn/post/7058211942510379039/)
- [Need simple kubectl command to see cluster resource usage #17512](https://github.com/kubernetes/kubernetes/issues/17512)
