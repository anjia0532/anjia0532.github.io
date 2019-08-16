
---

title: 008-Sentinel清洗RESTful的@PathVariable

date: 2019-03-05 18:30:00 +0800

tags: [spring boot,spring cloud,sentinel,hystrix,微服务,熔断]

categories: 微服务

---

> 这是坚持技术写作计划（含翻译）的第8篇，定个小目标999，每周最少2篇。


前段时间的文章多是运维方面的，最近放出一波后端相关的。

<a name="8e1b944f"></a>
## 背景
最近开始使用Sentinel进行流量保护，但是默认的web servlet filter是拦截全部http请求。在传统的项目中问题不大。但是如果项目中用了Spring MVC，并且用了@PathVariable就尴尬了。<br />比如 uri pattern是  `/foo/{id}` ,而从Sentinel监控看 `/foo/1` 和 `/foo/2` 就是两个资源了，并且Sentinel最大支持6000个资源，再多就不生效了。

<a name="81c1dff6"></a>
## 解决办法

<a name="beee100b"></a>
### 官方给的方案是:UrlCleaner

```java
 WebCallbackManager.setUrlCleaner(new UrlCleaner() {
            @Override
            public String clean(String originUrl) {
                if (originUrl.startsWith(fooPrefix)) {
                    return "/foo/*";
                }
                return originUrl;
            }
        });
```
但是想想就吐， `/v1/{foo}/{bar}/qux/{baz}` 这种的来个20来个，截一个我看看。

<a name="AOP"></a>
### AOP
换种思路，uri pattern难搞，用笨办法 aop总行吧？答案是可以的。

```java
@Aspect
public class SentinelResourceAspect {
    @Pointcut("within(com.anjia.*.web.rest..*)")
    public void sentinelResourcePackagePointcut() {
        // Method is empty as this is just a Pointcut, the implementations are
        // in the advices.
    }
    @Around("sentinelResourcePackagePointcut()")
    public Object sentinelResourceAround(ProceedingJoinPoint joinPoint) throws Throwable {
        Entry entry = null;
        // 务必保证finally会被执行
        try {
          // 资源名可使用任意有业务语义的字符串
          // 注意此处只是类名#方法名，方法重载是合并的，如果需要进行区分，
          // 可以获取参数类型加入到资源名称上
          entry = SphU.entry(joinPoint.getSignature().getDeclaringTypeName()+
                             "#"+joinPoint.getSignature().getName());
          // 被保护的业务逻辑
          // do something...
        } catch (BlockException ex) {
          // 资源访问阻止，被限流或被降级
          // 进行相应的处理操作
        } finally {
          if (entry != null) {
            entry.exit();
          }
        }
        return result;
    }
}
```


<a name="f7ae864d"></a>
### 拦截器
温习一下 Spring mvc的执行流程 `doFilter -> doService -> dispatcher -> preHandle -> controller -> postHandle -> afterCompletion -> filterAfter` 

核心的是 `String pattern = (String) request.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);` 但是是在dispatcher阶段才赋值的，所以在CommFilter是取不到的，所以导致使用官方的Filter是不行的。只能用拦截器

```java

import com.alibaba.csp.sentinel.EntryType;
import com.alibaba.csp.sentinel.SphU;
import com.alibaba.csp.sentinel.adapter.servlet.callback.RequestOriginParser;
import com.alibaba.csp.sentinel.adapter.servlet.callback.UrlCleaner;
import com.alibaba.csp.sentinel.adapter.servlet.callback.WebCallbackManager;
import com.alibaba.csp.sentinel.adapter.servlet.util.FilterUtil;
import com.alibaba.csp.sentinel.context.ContextUtil;
import com.alibaba.csp.sentinel.log.RecordLog;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.alibaba.csp.sentinel.util.StringUtil;
import org.apache.commons.lang3.StringUtils;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.HandlerMapping;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class SentinelHandlerInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String origin = parseOrigin(request);
        String pattern = (String) request.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
        String uriTarget = StringUtils.defaultString(pattern,FilterUtil.filterTarget(request));
        try {
            // Clean and unify the URL.
            // For REST APIs, you have to clean the URL (e.g. `/foo/1` and `/foo/2` -> `/foo/:id`), or
            // the amount of context and resources will exceed the threshold.
            UrlCleaner urlCleaner = WebCallbackManager.getUrlCleaner();
            if (urlCleaner != null) {
                uriTarget = urlCleaner.clean(uriTarget);
            }
            RecordLog.info(String.format("[Sentinel Pre Filter] Origin: %s enter Uri Path: %s", origin, uriTarget));
            SphU.entry(uriTarget, EntryType.IN);
            return true;
        } catch (BlockException ex) {
            RecordLog.warn(String.format("[Sentinel Pre Filter] Block Exception when Origin: %s enter fall back uri: %s", origin, uriTarget), ex);
            WebCallbackManager.getUrlBlockHandler().blocked(request, response, ex);
            return false;
        }
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        while (ContextUtil.getContext() != null && ContextUtil.getContext().getCurEntry() != null) {
            ContextUtil.getContext().getCurEntry().exit();
        }
        ContextUtil.exit();
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }

    private String parseOrigin(HttpServletRequest request) {
        RequestOriginParser originParser = WebCallbackManager.getRequestOriginParser();
        String origin = EMPTY_ORIGIN;
        if (originParser != null) {
            origin = originParser.parseOrigin(request);
            if (StringUtil.isEmpty(origin)) {
                return EMPTY_ORIGIN;
            }
        }
        return origin;
    }


    private static final String EMPTY_ORIGIN = "";
}

```

```java

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;

@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {
    @Inject
    SentinelHandlerInterceptor sentinelHandlerInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(sentinelHandlerInterceptor);
    }
}

```

<a name="999b84d8"></a>
## UrlBlockHandler和UrlCleaner和WebServletConfig.setBlockPage(blockPage)
上面说过，UrlCleaner是为了归并请求，清洗url用的。而UrlBlockHandler是在被拦截后的默认处理器。但是clean和handler都不是链式的，所以如果有多种处理，需要自己在一个方法里，进行逻辑判断。

UrlCleaner
```java
 WebCallbackManager.setUrlCleaner(new UrlCleaner() {
            @Override
            public String clean(String originUrl) {
                if (originUrl.startsWith(fooPrefix)) {
                    return "/foo/*";
                }
                return originUrl;
            }
        });
```

UrlBlockHandler<br />如果通用一点的，可以自己根据request的 content-type进行自适应返回内容(PLAN_TEXT和JSON)

```java
WebCallbackManager.setUrlBlockHandler((request, response, ex) -> {
    response.addHeader("Content-Type","application/json;charset=UTF-8");
    PrintWriter out = response.getWriter();
    out.print("{\"code\"":429,\"msg\":\"系统繁忙，请稍后重试\""}");
    out.flush();
    out.close();
});
```

WebServletConfig.setBlockPage(blockPage)
```java
WebServletConfig.setBlockPage("http://www.baidu.com")
```

注意，三个方法都不是不支持调用链，比如我写两个UrlBlockHandler,只认最后一个。

<a name="35808e79"></a>
## 参考资料

- [wiki/主流框架的适配#web-servlet](https://github.com/alibaba/Sentinel/wiki/主流框架的适配#web-servlet)
- [issues#](https://github.com/alibaba/Sentinel/issues/286)[REST API pattern UrlCleaner同一处理](https://github.com/alibaba/Sentinel/issues/286)

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。


