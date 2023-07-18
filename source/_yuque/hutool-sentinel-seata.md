---
title: 081-hutool httputil 集成 sentinel 和 seata
urlname: hutool-sentinel-seata
date: '2023-07-18 12:35:21 +0800'
tags:
  - hutool
  - sentinel
  - seata
categories:
  - java
  - hutool
---

> 这是坚持技术写作计划（含翻译）的第 81 篇，定个小目标 999 篇。

首先声明，不建议在生产中使用 hutool httputil 因为他没有线程池，性能较差，扩展性差。如果你已经用了 spring boot/ spring cloud 那可以用开箱即用的声明式 http 库，比如 [forest](https://gitee.com/dromara/forest) 或者 [feign](https://github.com/OpenFeign/feign) , 重度使用的话，可以用老牌的 [httpclient](https://github.com/apache/httpcomponents-client) 或者 [okhttp](https://github.com/square/okhttp) 等。

但是如果前期团队内部对此没有硬性要求，导致已经已经有历史包袱的情况下，又得上 sentinel 或者 seata 等框架。就得对其进行非侵入式改造了。

<!-- more -->

## Hutool 拦截器

Hutool 5.8.0.M2+ 以后增加了 `GlobalInterceptor.addRequestInterceptor、GlobalInterceptor.addResponseInterceptor` 和 `HttpRequest.addInterceptor` 参见 [使用 hutool 的 http 工具后，如何统一处理 Request 和 Response #2217](https://github.com/dromara/hutool/issues/2217)

### 记录 httputil 请求日志

```java


@Value("${hutool.http.print-log:false}")
private boolean hutoolHttpPrintLogEnabled;

// 打印 hutool http 请求和响应数据
if (hutoolHttpPrintLogEnabled) {
    GlobalInterceptor.INSTANCE.addRequestInterceptor((req) -> {
        log.info("hutool http 请求 url: {}, method: {}, headers: {}, form: {}, body: {}", req.getUrl(),
            req.getMethod().name(), req.headers(), req.form(), StrUtil.str(req.bodyBytes(), req.charset()));
    });
    GlobalInterceptor.INSTANCE.addResponseInterceptor((resp) -> {
        log.info("hutool http 响应 headers: {}, body: {}", resp.headers(),
            StrUtil.str(resp.bodyBytes(), resp.charset()));
    });
}
```

### 集成 seata

```java
@Value("${seata.enabled:false}")
private boolean seataEnabled;

// seata
if (seataEnabled) {
    GlobalInterceptor.INSTANCE.addRequestInterceptor((req) -> {
        req.header(RootContext.KEY_XID, RootContext.getXID());
    });
}
```

### 集成 sentinel

```java

@Value("${spring.cloud.sentinel.enabled:false}")
private boolean sentinelEnabled;

// sentinel
if (sentinelEnabled) {
    GlobalInterceptor.INSTANCE.addRequestInterceptor((req) -> {
        try {
            // 抹去参数，只保留 url
            Entry entry = SphU.entry(req.getMethod().name().toUpperCase() + ":"
                + StringUtils.substringBefore(req.getUrl(), "?"));
            ContextUtil.getContext().setCurEntry(entry);
        } catch (BlockException e) {
            log.error("Sentinel 触发流控快速失败", e);
            throw new ApiException("触发流控快速失败");
        }
    });
    GlobalInterceptor.INSTANCE.addResponseInterceptor((resp) -> {
            ContextUtil.getContext().getCurEntry().exit();
    });
}
```

然后自己写个 configuration 类，把代码塞进去就行了。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- 我的博客
- 我的掘金
