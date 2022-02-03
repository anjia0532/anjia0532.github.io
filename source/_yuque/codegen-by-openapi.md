---
title: 061-基于swagger/yapi/openapi逆向生成sdk
urlname: codegen-by-openapi
date: '2021-04-09 19:35:21 +0800'
tags:
  - java
  - 微服务
  - swagger
  - yapi
  - openapi
categories:
  - 微服务
---

> 这是坚持技术写作计划（含翻译）的第 61 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解如何基于三方提供的类 openapi 文档来逆向生成 sdk（少掉挨个手敲的烦恼），生成的 pojo 支持 lombok

<!-- more -->

## 前置准备

- Nodejs12 LTS+
- Java8+
- Git

研究了 swagger 自己的 [https://github.com/swagger-api/swagger-codegen](https://github.com/swagger-api/swagger-codegen)
微软家的 [https://github.com/Azure/autorest](https://github.com/Azure/autorest)
以及 openapi 的 [https://github.com/OpenAPITools/openapi-generator](https://github.com/OpenAPITools/openapi-generator)
最后选择了[https://github.com/OpenAPITools/openapi-generator](https://github.com/OpenAPITools/openapi-generator)

## 安装[openapi-generator](https://github.com/OpenAPITools/openapi-generator)

```bash
npm install @openapitools/openapi-generator-cli -g

git clone https://github.com/anjia0532/openapi-generator-templates

openapi-generator-cli generate -t openapi-generator-templates\generator-templates\JavaSpring\spring-boot-lombok-actuator -g spring -puseLombok=true -i swaggerApi.json --skip-validate-spec
```

参考资料 [https://github.com/OpenAPITools/openapi-generator#17---npm](https://github.com/OpenAPITools/openapi-generator#17---npm)
参考资料 [https://github.com/anjia0532/openapi-generator-templates/tree/master/generator-templates/JavaSpring/spring-boot-lombok-actuator](https://github.com/anjia0532/openapi-generator-templates/tree/master/generator-templates/JavaSpring/spring-boot-lombok-actuator)

## 生成结果

```bash
src
└─ main
   ├─ java
   │  └─ org
   │     └─ openapitools
   │        ├─ api           # 调用对方api的client
   │        │  ├─ xxx.java
   │        ├─ configuration # 配置相关
   │        │  ├─ xxx.java
   │        ├─ model         # pojo类
   │        │  ├─ xxx.java
   │        ├─ OpenAPI2SpringBoot.java
   │        └─ RFC3339DateFormat.java
   ├─ resources
   │  └─ application.properties
   │        └─ RFC3339DateFormat.java
   └─ pom.xml
```

以实际用到的某个不重要的 model 类演示下生成的效果

```java
package org.openapitools.model;

import java.util.Objects;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonCreator;
import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;
import org.openapitools.jackson.nullable.JsonNullable;
import javax.validation.Valid;
import javax.validation.constraints.*;

import lombok.*;

/**
 * EmptyObject2
 */
@javax.annotation.Generated(value = "org.openapitools.codegen.languages.SpringCodegen", date = "2021-04-09T19:10:24.687+08:00[Asia/Shanghai]")
@Getter
@Setter
@Builder(toBuilder = true)
@NoArgsConstructor
@AllArgsConstructor
@ToString
public class EmptyObject2   {
        /**
         * 获取渠道
         */
        @JsonProperty("getChannel")
        private String getChannel;

        /**
         * 优惠券ID
         */
        @JsonProperty("couponIds")
        private String couponIds;

        /**
         * 会员ID
         */
        @JsonProperty("memberId")
        private String memberId;

}
```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/04/09/codegen-by-openapi)
- [我的掘金](https://juejin.cn/post/6949114011195555853/)
