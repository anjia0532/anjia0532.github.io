---
title: ansible
date: 2018-04-28 13:23:28
tags: [linux,ansible,运维,python]
---



## 环境





## 安装

参考资料 [官方Installation Guide](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#id8)  [中文Installation](http://www.ansible.com.cn/docs/intro_installation.html#id9)

```bash
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible -y
$ sudo apt-get update
$ sudo apt-get install ansible
```

查看安装结果

```bash
$ ansible --version
ansible 2.4.3.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/xxx/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python2.7/dist-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 2.7.12 (default, Dec  4 2017, 14:50:18) [GCC 5.4.0 20160609]
```



## 







参考 [Linux轻量级自动运维工具-Ansible浅析](http://blog.51cto.com/weiweidefeng/1895261)