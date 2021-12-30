---
title: 054-微服务之真正的优雅停机
urlname: micro-elegant-shutdown
date: 2021-01-18 23:35:21 +0800
tags: [微服务,springboot,springcloud,nacos,eureka]
categories: [微服务]
---

> 这是坚持技术写作计划（含翻译）的第 54 篇，定个小目标 999，每周最少 2 篇。

网上一些大佬主要集中在如何从负载均衡里安全的把实例摘除（[🔥Serverless 微服务优雅关机实践| 🏆 技术专题第七期征文](https://juejin.cn/post/6895340763208089614)）

但是实际上你会使用\_shutdown 断点去停掉实例么？

使用 apollo/naocs/eureka 等注册中心来操作节点上下线会更优雅一些

对比 Shutdown Hook 来看，shutdown hook 的初衷是关机之前做一些善后工作（从注册中心/slb 摘除自己，关掉消息队列等），但是假如我只是想切换流量而不关机（比如发版时只切流量，另外版本继续运行待命，有问题流量再切回去），使用 shutdown hook 钩子就不合适了

本文主要讲解 shutdown hook + nacos 上下线事件实现的真正的优雅停机。

<!-- more -->

以 nacos 为例，项目启动时，监听 nacos 服务实例变更事件
需要判断当前实例有没有发生变化（主要是是否上下线，如果需要判断权重或者元数据，需要自行修改，此处不提供），如果没有变化认为与己无关，丢弃就可以了

如果发生变化后，使用 `SpringUtil.publishEvent`  广播 Spring Event，这样项目内其他地方，包括但不限于消息队列，定时任务（比如 xxljob）等就可以自行 `@EventListener(NacosEvent.class)`  监听即可

```java

import com.alibaba.nacos.api.naming.pojo.Instance;
import lombok.Data;
import lombok.experimental.Accessors;
import org.springframework.context.ApplicationEvent;

import java.util.Map;

/**
 * Nacos事件对象
 *
 * @author AnJia
 */
public class NacosEvent extends ApplicationEvent {

    /**
     * Create a new {@code ApplicationEvent}.
     *
     * @param source the object on which the event initially occurred or with
     *               which the event is associated (never {@code null})
     */
    public NacosEvent(NacosModel source) {
        super(source);
    }

    /**
     * Nacos事件对象
     *
     * @author AnJia
     */
    @Data
    @Accessors(chain = true)
    public static class NacosModel {
        /**
         * 当前服务可用列表
         */
        Map<String, Instance> instanceMap;
        /**
         * 本实例的实例id
         */
        private String instanceId;
        /**
         * 本实例可用还是不可用
         */
        private Boolean enable;
        /**
         * 是否有变更
         */
        private boolean changed;
    }
}
```

```java

import com.alibaba.cloud.nacos.NacosDiscoveryProperties;
import com.alibaba.nacos.api.exception.NacosException;
import com.alibaba.nacos.api.naming.listener.NamingEvent;
import com.alibaba.nacos.api.naming.pojo.Instance;
import com.shunzhongkeji.util.SpringUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;
import java.util.HashMap;
import java.util.Map;

@Slf4j
@Configuration
public class NacosConfiguration {
    private final NacosDiscoveryProperties nacosDiscoveryProperties;

    private final String serviceName;
    private static boolean ENABLE = Boolean.FALSE;

    public NacosConfiguration(NacosDiscoveryProperties nacosDiscoveryProperties,
                              @Value("${spring.application.name}") String serviceName) {
        this.nacosDiscoveryProperties = nacosDiscoveryProperties;
        this.serviceName = serviceName;
    }


    /**
     * 项目初始化时监听nacos上下线
     *
     * @throws NacosException nacos报错
     */
    @PostConstruct
    public void init() throws NacosException {
        nacosDiscoveryProperties.namingServiceInstance().subscribe(serviceName, event -> {
            Map<String, Instance> instanceMap = new HashMap<>(16);
            if (event instanceof NamingEvent) {
                ((NamingEvent) event).getInstances()
                    .forEach(instance -> instanceMap.put(instance.getInstanceId(), instance));
            }
            // nacos 实例id，示例 192.168.40.65#9090#DEFAULT#DEFAULT_GROUP@@oms
            String instanceId =
                nacosDiscoveryProperties.getIp() + "#"
                    + nacosDiscoveryProperties.getPort() + "#"
                    + nacosDiscoveryProperties.getClusterName() + "#"
                    + nacosDiscoveryProperties.getGroup() + "@@"
                    + nacosDiscoveryProperties.getService();
            boolean changed = (ENABLE != instanceMap.containsKey(instanceId));
            ENABLE = instanceMap.containsKey(instanceId);

            NacosEvent.NacosModel nacosEvent = new NacosEvent.NacosModel()
                .setEnable(ENABLE)
                .setInstanceId(instanceId)
                .setInstanceMap(instanceMap)
                .setChanged(changed);

            SpringUtil.publishEvent(new NacosEvent(nacosEvent));
        });
    }
}
```

```java

@Slf4j
@Configuration
public class RabbitConfiguration {

    private final RabbitListenerEndpointRegistry registry;

    public RabbitConfiguration(RabbitListenerEndpointRegistry registry) {
        this.registry = registry;
    }

    /**
     * MQ 监听nacos上下线事件
     *
     * @param event nacos上下线事件
     */
    @Async
    @EventListener(NacosEvent.class)
    public void healthEventChange(NacosEvent event) {
        NacosModel model = (NacosModel) event.getSource();
        if (model.isChanged()) {
            if (Boolean.TRUE.equals(model.getEnable())) {
                registry.start();
                log.info("{}实例上线,启用rabbit监听", model.getInstanceId());
            } else {
                registry.stop();
                log.info("{}实例下线,禁用rabbit监听", model.getInstanceId());
            }
        }
    }
}
```

关于 ShutDownHook 部分就不写了，别人写的太多了，可以参考 [Spring 优雅关闭之：ShutDownHook](https://blog.csdn.net/qq_26323323/article/details/89814410)

这样就可以结合 jenkins/gitlab ci 等 CI/CD 工具实现，优雅切流量/停机了，比如发布新版本后，可以默认让新版本是下线状态，这时一点流量没有（定时任务，外部流量，消息队列都没有），扩容等操作完成后（ready 状态），再上线，此时流量会同时到达新老版本，发现新版本有问题，可以把新版本下线，修复后重新发版，如果新版本没问题，可以把老版本下掉，并行运行一段时间（比如 1 小时），确定没问题后，老版本直接断电就行了 。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/01/18/micro-elegant-shutdown)
- [我的掘金](https://juejin.cn/post/6919054913142816776/)
- [🔥Serverless 微服务优雅关机实践| 🏆 技术专题第七期征文](https://juejin.cn/post/6895340763208089614)
- [Spring 优雅关闭之：ShutDownHook](https://blog.csdn.net/qq_26323323/article/details/89814410)
