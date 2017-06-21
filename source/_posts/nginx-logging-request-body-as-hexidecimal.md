---
title: nginx日志中$request_body 十六进制字符(\x22\x9B\x5C\x09\x08...)完美解决方案
date: 2017-06-21 10:11:46
tags: [nginx,logstash,elk,elkstack]
categories: [nginx]
---
在使用nginx记录访问日志时，发现在含有`request_body`的 `PUT`,`POST` 请求时，日志中会含有 `\x22``\x9B``\x5C``\x09``\x08` 字符，不利于阅读和处理。

<!-- more -->

具体 支持`request_body`的http method参见 [http1.1定义 9 Method Definitions][link9MethodDefinitions] 和 [Payloads of HTTP Request Methods][linkPayloadsOfHttpRequestMethods]


`nginx.conf` 默认access_log 配置
```
    log_format  main '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"'
                      '$http_host $upstream_status $upstream_addr $request_time $upstream_response_time'; 
```

改成
```
    log_format json_log escape=json '{"realip":"$remote_addr","@timestamp":"$time_iso8601","host":"$http_host","request":"$request","req_body":"$request_body","status":"$status","size":$body_bytes_sent,"ua":"$http_user_agent","cookie":"$http_cookie","req_time":"$request_time","uri":"$uri","referer":"$http_referer","xff":"$http_x_forwarded_for","ups_status":"$upstream_status","ups_addr":"$upstream_addr","ups_time":"$upstream_response_time"}';
```

参考 [How to generate a JSON log from nginx?][linkHowToGenerateAJsonLogFromNginx?]

[官方文档ngx_http_log_module.html#log_format][ngx_http_log_module] 注意，`escape`是从*1.11.8*后新增的参数。

如果是老版本的，linux可以考虑使用`shell`命令替换，`logstash`可以考虑使用`ruby`处理 ，参考 [Optionally support handling of \x escape codes][linkOptionallySupportHandlingOf\xEscape]

[link9MethodDefinitions]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html
[linkPayloadsOfHttpRequestMethods]: https://stackoverflow.com/questions/5905916/payloads-of-http-request-methods#answer-5928241
[linkHowToGenerateAJsonLogFromNginx?]: https://stackoverflow.com/questions/25049667/how-to-generate-a-json-log-from-nginx#answer-42564710
[ngx_http_log_module]: http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format
[linkOptionallySupportHandlingOf\xEscape]: https://github.com/logstash-plugins/logstash-codec-json/issues/2
