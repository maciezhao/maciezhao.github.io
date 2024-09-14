---
title: blog home
date: 2024-09-14 11:31:12
tags:
---

# 关于SpringBoot 3.x Context Propagation的不同

Spring Boot 3.x 将sleuth大部分实现迁移到Micrometer-Tracing中，链路追踪发生了一些小变化

在Spring Boot 2.x中，默认是使用X-B3-TraceId, X-B3-ParentSpanId, X-B3-SpanId http头来传递链路id

服务A调用服务B，服务B直接使用头中的数据，包括X-B3-SpanId，这里的X-B3-SpanId是服务A在发起调用之前生成的新spanId, 原spanId将放到X-B3-ParentSpanId头中

这里主要涉及到sleuth中`spring.sleuth.supportsJoin`的配置，默认为true, 所以服务B会将头中的值直接设置称当前值，参考：[https://github.com/openzipkin/brave/tree/master/brave#sharing-span-ids-between-client-and-server](https://github.com/openzipkin/brave/tree/master/brave#sharing-span-ids-between-client-and-server)

示例：

服务A:

```
# traceId, parentSpanId, spanId
[9eef579b511cf67f][][9eef579b511cf67f]
```

发起调用的请求头：

```
X-B3-TraceId: 9eef579b511cf67f
X-B3-ParentSpanId: 9eef579b511cf67f
X-B3-SpanId: 25116cbcdf747770
```

服务B:

```
# traceId, parentSpanId, spanId
[9eef579b511cf67f][9eef579b511cf67f][25116cbcdf747770]
```

在Spring Boot 3.x中，默认采用W3C实现：

springboot 3 默认不将parentId的值写入MDC中，如果需要显示parentId的值，需要注入以下bean:

```java
    @Bean
    CorrelationScopeCustomizer parentIdCorrelationScopeCustomizer() {
        return builder -> builder.add(CorrelationScopeConfig.SingleCorrelationField.create(BaggageFields.PARENT_ID));
    }
```

服务A:

```
# traceId, parentId, spanId
[664ef63afebd8cf8938111d0ed6f53ba][938111d0ed6f53ba][89b7d1512d122e2f]
```

请求头：

```json
traceparent: 00-664ef63afebd8cf8938111d0ed6f53ba-89b7d1512d122e2f-00
```

服务B:

```
# traceId, parentId, spanId
[664ef63afebd8cf8938111d0ed6f53ba][89b7d1512d122e2f][49b23875df5f1e03]
```

664ef63afebd8cf8938111d0ed6f53ba为traceId, 89b7d1512d122e2f是本服务的spanId, 当调用下游服务时，下游服务traceId保持不变，spanId会重新生成，请求头中的spanId将会当作parentId(即父spanId)传递

若要Spring Boot 2.x和3.x行为保持一致，需要在Spring Boot 2.x中设置supportsJoin为false

```java
spring.sleuth.supportsJoin=false
```

若要Spring Boot 3.x支持supportsJoin,就不能使用W3C格式，必须使用brave支持的B3/B3_MULTI格式：

```java
management.tracing.propagation.produce=B3
management.tracing.propagation.consume=B3
management.tracing.brave.span-joining-supported=true
```

参考资料：

[https://stackoverflow.com/questions/77389421/spring-micrometer-tracing-not-honoring-parentspanid-or-converting-x-b3-spanid](https://stackoverflow.com/questions/77389421/spring-micrometer-tracing-not-honoring-parentspanid-or-converting-x-b3-spanid)

[https://github.com/micrometer-metrics/tracing/wiki/Spring-Cloud-Sleuth-3.1-Migration-Guide#async-instrumentation](https://github.com/micrometer-metrics/tracing/wiki/Spring-Cloud-Sleuth-3.1-Migration-Guide#async-instrumentation)

[https://github.com/openzipkin/brave/tree/master/brave#sharing-span-ids-between-client-and-server](https://github.com/openzipkin/brave/tree/master/brave#sharing-span-ids-between-client-and-server)

使用supportsJoin，有一些不兼容