---
title: 004-零失败快速搞定通配符SSL证书
urlname: lets-encrypt-acme-sh
date: 2019-02-18 12:43:00 +0800
tags: [ssl,https,http2,lets-encrypt,acme,acme-sh]
categories: [运,维]
---

> 这是坚持技术写作计划（含翻译）的第四篇，定个小目标 999，每周最少 2 篇。

> 过去几年中，我们一直主张站点采用 HTTPS，以提升其安全性。去年的时候，我们还通过将更大的 HTTP 页面标记为‘不安全’以帮助用户。
> 不过从 2018 年 7 月开始，随着 Chrome 68 的发布，浏览器会将所有 HTTP 网站标记为‘不安全’。
> 引用自 [chrome 68 发布说明](https://support.google.com/chrome/a/answer/7679408)

得益于 Google 等大厂的消灭 HTTP 运动和[Let's Encrypt](https://letsencrypt.org)  非盈利组织的努力，越来越多的站点开始迁移到 HTTPS，下图是[Let's Encrypt 的统计数据](https://letsencrypt.org/stats/)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1550317966768-d4587466-a6d0-4868-b9ca-4e65bc45b101.png#align=left&display=inline&height=450&name=image.png&originHeight=450&originWidth=940&size=41971&width=940)

<!-- more -->

## 什么是 Let's Encrypt

部署 HTTPS 网站的时候需要证书，证书由 CA 机构签发，大部分传统 CA 机构签发证书是需要收费的，这不利于推动 HTTPS 协议的使用。

Let's Encrypt 是一个国外的非盈利的 CA 证书机构，旨在以自动化流程消除手动创建和安装证书的复杂流程，并推广使万维网服务器的加密连接无所不在，为安全网站提供免费的 SSL/TLS 证书。

由 Linux 基金会托管，许多国内外互联网大厂都对其进行赞助，目前主流浏览器均已信任 Let's Encrypt 发放的证书。

注意，Let's Encrypt 颁发的都是 DV 证书，不提供 OV,EV 证书。

本文主要讲解 如何使用 Let's Encrypt 颁发通配符证书。

## 通配符证书

通配符 SSL 证书旨在保护主域名以及旗下不限数量的子域，即用户可通过单个通配符 SSL 证书可保护任意数量的子域。如果用户拥有多个子域名平台，可通过通配符 SSL 证书保护这些子域名。

但是目前 Let's Encrypt 只支持同级子域名通配符。例如 `*.demo.com`  只支持 `xx.demo.com`  这种的，而不支持 `xx.xx.demo.com` ，而要支持二级通配符，需要再次颁发二级通配符证书 类似  `*.demo.demo.com` ，注意，这种的二级通配符，要求，一级域名是固定的，意即，不支持 `*.*.demo.com`

## 使用  acme.sh 简化证书颁发操作

官方建议使用[Certbot](https://certbot.eff.org/) ，但是很长一段时期 Certbot 不支持通配符（现在已经支持），而且对于证书自动续期支持的也不好。并且操作时也挺麻烦。

### 安装 acme.sh

```bash
$ curl https://get.acme.sh | sh
# 或者
$ wget -O -  https://get.acme.sh | sh
# 或者
$ git clone https://github.com/Neilpang/acme.sh.git
$ ./acme.sh/acme.sh --install
```

### DNS Api 颁发通配符证书

acme.sh 功能很强大，此处只介绍使用 Dns Api 自动化颁发通配符证书. 目前支持包含阿里和 DNSPod 在内的 60 家 dns 服务商（参见 [Currently acme.sh supports](https://github.com/Neilpang/acme.sh#currently-acmesh-supports)）

如果你的 DNS 服务商不提供 API 或者 acme.sh 暂未支持，或者处于安全方面的考虑，不想将重要的域名的 API 权限暴露给 acme.sh，可以申请一个测试域名，然后在重要域名上设置 CNAME（参见 [DNS alias mode](https://github.com/Neilpang/acme.sh/wiki/DNS-alias-mode)）

假设您的域名在 DNSPod 托管，登陆 DNSPod 后台，依次打开 用户中心->安全设置-> API Token->查看->创建 API Token-> 输入任意 token 名称->确定-> 保存 ID 和 Token 值（图中打码部分）

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1550463481824-01a645a7-5a95-4320-80ab-753ff7664bff.png#align=left&display=inline&height=558&name=image.png&originHeight=558&originWidth=1241&size=59891&width=1241)

```bash
$ export DP_Id="你的ID"
$ export DP_Key="你的Token"
$ acme.sh --issue --dns dns_dp -d example.com -d *.example.com
# 如果 使用了DNS别名，还需要增加 --challenge-alias 别名域名 参数
# 为了防止dns不生效，脚本会暂停2分钟，并倒计时(Sleep 120 seconds for the txt records to take effect),等待即可
# 如果成功会出现 Cert success. 字样
# 不建议直接用~/.acme.sh 下的证书，参考 https://github.com/Neilpang/acme.sh/wiki/说明#3-copy安装-证书
# 需要使用 --installcert 复制到指定目录
$ acme.sh --installcert  \
				-d  example.com -d *.example.com \
        --key-file /etc/letsencrypt/live/example.com/privkey.pem \
        --fullchain-file /etc/letsencrypt/live/example.com/fullchain.pem \
        --reloadcmd  "service nginx reload"
```

### 优化 HTTPS 配置

本文以  [Mozilla SSL Configuration Generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/)  生成的 nginx 为例，同样也可以生成 Apache 和 IIS

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    # Redirect all HTTP requests to HTTPS with a 301 Moved Permanently response.
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    # certs sent to the client in SERVER HELLO are concatenated in ssl_certificate
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
    # openssl dhparam -out /etc/letsencrypt/live/example.com/ 2048
    ssl_dhparam /etc/letsencrypt/live/example.com/dhparam.pem;

    # intermediate configuration. tweak to your needs.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;

    ## verify chain of trust of OCSP response using Root CA and Intermediate certs
    # ssl_trusted_certificate /path/to/root_CA_cert_plus_intermediates;

    resolver <IP DNS resolver>;

    ....
}
```

### 检查 HTTPS 得分

访问  [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)  提交自己域名，进行评分

## 参考资料

- [Chrome 将不再标记 HTTPS 页面为安全站点](https://www.oschina.net/news/96200/chrome-will-stop-tag-https-site-secure)
- [Let's Encrypt Stats](https://letsencrypt.org/stats/)
- [About Let's Encrypt](https://letsencrypt.org/about/)
- [Mozilla SSL Configuration Generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/)
- [DV 型、OV 型、EV 型证书的主要区别](https://www.cnblogs.com/sslwork/p/6193256.html)
- [Neilpang/acme.sh#Wiki#安装说明](https://github.com/Neilpang/acme.sh/blob/master/dnsapi/README.md)
- [Neilpang/acme.sh#Wiki#How to Install](https://github.com/Neilpang/acme.sh/wiki/How-to-install)
- [Neilpang/acme.sh#Wiki#How to use DNS API](https://github.com/Neilpang/acme.sh/wiki/How-to-install)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
