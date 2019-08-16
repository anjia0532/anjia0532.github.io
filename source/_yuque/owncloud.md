---
title: 自建私有云Owncloud+Nginx（支持16G大文件上传）
date: 2017-04-05 12:21:25
tags: [owncloud]
categories: [owncloud]
---

## [Owncloud官网](https://owncloud.org/)
桌面版支持Windows,Mac,Linux 移动版本支持，android,ios,blackberry

## 环境
- Ubuntu-16.04_64
- Owncloud9.14-2.1
- SQLite3
- PHP7
- Nginx 1.10.0

## 最简单安装

### 根据linux版本选择相应版本
[owncloud-9.1](http://download.owncloud.org/download/repositories/9.1/owncloud/)

### 安装
以Ubuntu-16.04 安装owncloud-9.14-2.1为例

#### 用root权限添加owncloud密钥

```bash
su root

wget -nv https://download.owncloud.org/download/repositories/9.1/Ubuntu_16.04/Release.key -O Release.key
apt-key add - < Release.key
```

#### 用root权限添加owncloud软件源
```bash
sh -c "echo 'deb http://download.owncloud.org/download/repositories/9.1/Ubuntu_16.04/ /' > /etc/apt/sources.list.d/owncloud.list"
apt  update -y && apt install owncloud -y
```

## 源码安装

### 安装PHP7

```bash
sudo apt-get install -y php7.0-common  php7.0-gd php7.0-json php7.0-mysql php7.0-curl  php7.0-intl php7.0-mcrypt php-imagick  php7.0-zip php7.0-xml php7.0-mbstring
```

### 安装数据库
```bash
#mariadb
sudo apt-get install -y mariadb-server php7.0-mysql

#sqlite3
sudo apt-get install -y sqlite3 php7.0-sqlite3
```

### 安装web容器
```bash
#apache2
sudo apt-get install -y apache2 libapache2-mod-php7.0

#nginx
sudo apt-get install -y nginx php7.0-fpm 
```

### 修改fpm配置文件(nginx)
```bash
$ vi /etc/php/7.0/fpm/pool.d/www.conf
```

修改`listen = /run/php/php7.0-fpm.sock`为`listen=127.0.0.1:9000`(大约36行)

放开`env`的注释(大约384-388行)
```
env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
```


### 下载最新源码
```bash
$ wget -P /tmp https://download.owncloud.org/download/community/owncloud-latest.zip  && sudo unzip /tmp/owncloud-latest.zip -d /var/www/ && rm -rf /tmp/owncloud-latest.zip
```

### 给www-data授权
```bash
sudo chown -R www-data:www-data /var/www/owncloud/
```

### 参考资料
[官方nginx+https配置](https://doc.owncloud.org/server/9.1/admin_manual/installation/nginx_examples.html)

[支持大文件上传(16G)](https://github.com/owncloud/documentation/wiki/Uploading-files-up-to-16GB)

### 我的nginx配置

#### nginx
```bash
$ vi  /etc/nginx/sites-enabled/owncloud.conf

upstream php-handler {
    server 127.0.0.1:9000;
    #server unix:/var/run/php5-fpm.sock;
}

server {
    listen 10010;
    server_name 127.0.0.1;

    # Add headers to serve security related headers
    # Before enabling Strict-Transport-Security headers please read into this topic first.
    #add_header Strict-Transport-Security "max-age=15552000; includeSubDomains";
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header X-Download-Options noopen;
    add_header X-Permitted-Cross-Domain-Policies none;

    # Path to the root of your installation
    root /var/www/owncloud/;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # The following 2 rules are only needed for the user_webfinger app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
    #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json last;

    location = /.well-known/carddav {
        return 301 $scheme://$host/remote.php/dav;
    }
    location = /.well-known/caldav {
        return 301 $scheme://$host/remote.php/dav;
    }

    location /.well-known/acme-challenge { }

    # set max upload size
    client_max_body_size 16400M;
    fastcgi_buffers 64 4K;
    fastcgi_read_timeout 600;
    client_body_buffer_size 1048576k;
    client_body_temp_path /tmp/owncloud;
    
    # Disable gzip to avoid the removal of the ETag header
    gzip off;

    # Uncomment if your server is build with the ngx_pagespeed module
    # This module is currently not supported.
    #pagespeed off;

    error_page 403 /core/templates/403.php;
    error_page 404 /core/templates/404.php;

    location / {
        rewrite ^ /index.php$uri;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
        return 404;
    }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
        return 404;
    }

    location ~ ^/(?:index|remote|public|cron|core/ajax/update|status|ocs/v[12]|updater/.+|ocs-provider/.+|core/templates/40[34])\.php(?:$|/) {
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        #fastcgi_param HTTPS on;
        fastcgi_param modHeadersAvailable true; #Avoid sending the security headers twice
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off; #Available since nginx 1.7.11
    }

    location ~ ^/(?:updater|ocs-provider)(?:$|/) {
        try_files $uri $uri/ =404;
        index index.php;
    }

    # Adding the cache control header for js and css files
    # Make sure it is BELOW the PHP block
    location ~* \.(?:css|js)$ {
        try_files $uri /index.php$uri$is_args$args;
        add_header Cache-Control "public, max-age=7200";
        # Add headers to serve security related headers (It is intended to have those duplicated to the ones above)
        # Before enabling Strict-Transport-Security headers please read into this topic first.
        #add_header Strict-Transport-Security "max-age=15552000; includeSubDomains";
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header X-Download-Options noopen;
        add_header X-Permitted-Cross-Domain-Policies none;
        # Optional: Don't log access to assets
        access_log off;
    }

    location ~* \.(?:svg|gif|png|html|ttf|woff|ico|jpg|jpeg)$ {
        try_files $uri /index.php$uri$is_args$args;
        # Optional: Don't log access to other assets
        access_log off;
    }
}
```
#### php.ini
```bash

$ sudo vi /etc/php/7.0/fpm/php.ini

##修改以下几个配置参数

; should be bit bigger than upload_max_filesize 16400M = 16G + 16M = 16 * 1025 MB
post_max_size = 16400M

; cannot be bigger than post_max_size
upload_max_filesize = 16G

; on online servers this could require bigger values (my server is at home)
max_input_time = 3600

; from ownCloud documentation - not sure if is required
output_buffering = Off

; not sure if it is required [3] but it seems like ownCloud needs time to move the file to it's
; final place after upload and that can take quite some time for big files
max_execution_time = 1800

; you may also want to point this to a folder having enough space for big files being uploaded
upload_tmp_dir = /tmp/owncloud

```


### 启动服务
```bash
$ sudo service php7.0-fpm restart

$ sudo service nginx restart
```

### 配置
浏览器打开`http://127.0.0.1:10010`,MariaDB是Mysql的开源分支(mysql被oracle收购了)，适合大规模使用，对并发和性能要求比较高的场景。SQLite3适合小规模使用。此处使用SQLite3。详见 https://doc.owncloud.org/server/latest/admin_manual/configuration_database/db_conversion.html 和https://doc.owncloud.org/server/latest/admin_manual/configuration_database/linux_database_configuration.html

![owncloud.png](https://user-gold-cdn.xitu.io/2017/4/5/4469d32ea305747b364c60c6f03d8a39.png)

### 配置域名
详见 https://doc.owncloud.org/server/latest/admin_manual/configuration_server/config_sample_php_parameters.html
```bash
sudo vi /var/www/owncloud/config/config.php
```
修改
```
'trusted_domains' => 
  array (
    0 => '127.0.0.1:10010',
    1 => '域名',
  ),
```

修改
```
'overwrite.cli.url' => 'http://域名',
```

### 创建用户

浏览器访问`http://127.0.0.1:10010/settings/users`,用管理员用户名密码登陆

### 下载客户端

参见 https://owncloud.org/install/#install-clients