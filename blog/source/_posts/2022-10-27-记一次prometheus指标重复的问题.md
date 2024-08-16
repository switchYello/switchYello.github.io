---
layout:     post
title:      "记一次prometheus指标重复的问题"
subtitle:   ""
date:       2022-10-27
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- Spring
- Spock
- 单元测试
typora-root-url: ..
---


# 记一次prometheus指标重复的问题

​	

使用的reslience4j做熔断器，使用prometheus统计熔断器的指标监控，其中有一项指标是访问次数，用来绘制tps折线图。后来发现访问一次会导致指标增加两次。

​	

​		使用如下方式配置熔断器，并将其绑定到micrometer上。

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;

@Configuration
public class CircuitConfig {

    //micrometer提供,本质上底层还是代理到prometheus上
    @Resource
    private MeterRegistry meterRegistry;

    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        CircuitBreakerRegistry cr = CircuitBreakerRegistry.of(circuitConfig());
        TaggedCircuitBreakerMetrics.ofCircuitBreakerRegistry(cr).bindTo(meterRegistry);
        return cr;
    }
```



​	同时系统引入了spring.cloud的相关组件，其中有一段是这样的。注意看红圈地方，有做了一次绑定，其中的factory中获取到的CircuitBreakerRegistry就是上面代码中配置的CircuitBreakerRegistry。也就是该CircuitBreakerRegistry绑定了两次MeterRegistry。

​	这也就导致了发生一次调用，指标统计两次。

![image-20221027191017888](/img/in/2022-10-27-记一次prometheus指标重复的问题/image-20221027191017888.png)





## 总结

​	因为springboot的组件，检测到我们相应类存在时，会自动装配。其中就包含将熔断器绑定到指标监控的参数，而我又做了一次，导致的该问题，后续我将自己绑定的部分去掉了。