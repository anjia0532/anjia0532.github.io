---
title: gitlab迁移到docker并升级大版本到10.1.1和汉化
date: 2017-11-07 10:38:37
tags: [gitlab,docker,docker-compose]
---

本文主要讲 gitlab切换为docker版本，并且升级大版本(9.x-10.x)的较为快捷的方式

<!--more-->

## gitlab备份

### 查看现有版本
```
sudo gitlab-rake gitlab:env:info

...
GitLab information
Version:    9.2.5
...
```

### 备份
在原服务器运行
```
sudo gitlab-rake gitlab:backup:create RAILS_ENV=production

sudo sh -c 'umask 0077; tar -cf /var/opt/gitlab/backups/$(date "+etc-gitlab-%s_%Y_%m_%d.tar") -C /etc/gitlab'
```

通过`sudo ls -lah /var/opt/gitlab/backups | grep $(date "+%Y_%m_%d" )` 查看

```
-rw-------  1 git  git  172M 11月  7 11:07 1510024070_2017_11_07_x.x.x_gitlab_backup.tar
-rw-------  1 root root 150K 11月  7 11:28 etc-gitlab-1510025309_2017_11_07.tar
```

### 移动到目标服务器

使用`scp`将备份文件复制到目标主机

`username`是用户名
`ip`是来源主机ip

登陆目标主机，

```bash
sudo mkdir -p /data/gitlab/data/backups

scp username@ip:/var/opt/gitlab/backups/1510024070_2017_11_07_x.x.x_gitlab_backup.tar /data/gitlab/data/backups/1510024070_gitlab_backup.tar
scp username@ip:/var/opt/gitlab/backups/etc-gitlab-1510025309_2017_11_07.tar /data/gitlab/data/backups/

# 需要注意ssh的权限问题，如果无权限，要么改配置，要么就用winscp,ftp等进行上传
```

## gitlab恢复

### docker-compose
```yaml
version: '2'
services:
    gitlab:
      image: 'gitlab/gitlab-ce:x.x.x-ce.0' # 将x.x.x-ce.0改成之前gitlab版本,否则无法恢复备份
      restart: unless-stopped
      ports:
        - '80:80'
        - '443:443'
        - '22:22'
      volumes:
        - config:/etc/gitlab
        - data:/var/opt/gitlab
        - logs:/var/log/gitlab
volumes:
    config:/data/gitlab/config
    data:/data/gitlab/data
    logs:/data/gitlab/log
```

`docker-compose up -d`

### 恢复数据

```bash
docker exec -it gitlab_gitlab_1 /bin/bash

gitlab-rake gitlab:backup:restore RAILS_ENV=production BACKUP=1510024070 # 1510024070_gitlab_backup.tar 的前段
tar -xf /var/opt/gitlab/backups/etc-gitlab-1510025309_2017_11_07.tar -C /

```

访问以下http://ip/如果正常，则执行`docker-compose down`

## gitlab升级和汉化

```yaml
version: '2'
services:
    gitlab:
      image: 'anjia0532/gitlab-ce-zh:10.1.1-ce.0' # 汉化的10.1.1版本
      restart: unless-stopped
      ports:
        - '80:80'
        - '443:443'
        - '22:22'
      volumes:
        - config:/etc/gitlab
        - data:/var/opt/gitlab
        - logs:/var/log/gitlab
volumes:
    config:/data/gitlab/config
    data:/data/gitlab/data
    logs:/data/gitlab/log
```

参考连接:

- [gitlab服务器迁移--深山鬼怪][]

- [Gitlab CE 8.9 升级/迁移到GitLab CE 9.3.4 -- baowei][GitlabCe8.9升级/迁移到gitlabCe9.3.4--]

- [Backups -- gitlab-ce-doc][Backups--Gitlab-ce-doc]

博客 [https://anjia.ml/2017/11/07/gitlab-upgrade/][blog]
掘金 [https://juejin.im/post/5a0170a9f265da430702aea5][juejin]
简书 [http://www.jianshu.com/p/3ac4bd8372e0][jianshu]

[blog]: https://anjia.ml/2017/11/07/gitlab-upgrade/
[juejin]: https://juejin.im/post/5a0170a9f265da430702aea5
[jianshu]: http://www.jianshu.com/p/3ac4bd8372e0
[gitlab服务器迁移--深山鬼怪]: http://www.cnblogs.com/wenwei-blog/p/6362829.html
[GitlabCe8.9升级/迁移到gitlabCe9.3.4--]: http://www.jianshu.com/p/79447d5bf99e
[Backups--Gitlab-ce-doc]: https://docs.gitlab.com/omnibus/settings/backups.html
