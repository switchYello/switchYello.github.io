---
layout:     post
title:      "springGateway配置需要注意的点"
subtitle:   ""
date:       2020-09-09
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springGateway
- spring
typora-root-url: ..
---



# springGateway配置需要注意的点



## 1. 超时时间

​		springGateway使用的是WebFlux里面的WebClient来执行请求，WebClient有一些超时时间需要配置，

默认的可能不太合适。

```properties
#默认连接时间，默认45秒，太长了可以在此配置
spring.cloud.gateway.httpclient.connect-timeout=
#响应读取时间，默认不设置超时时间，可以在此设置
spring.cloud.gateway.httpclient.response-timeout=
#HttpClient连接池，最大空闲时间，默认无限制，可以在此配置
spring.cloud.gateway.httpclient.pool.max-idle-time=
```





## 2.缓存时间

​			gateway里面难免要使用负载均衡，如果使用的是Spring-cloud-loadbalance实现，loadbalance从discover-client拿到实例信息会存入缓存，默认缓存30秒。因为discover-client本身可能也带有缓存(不同的discover-client可能不一样)，所以loadbalance里的缓存没必要太长时间，此处调整成5秒

```properties
#默认30s，但其实没必要这么长时间
spring.cloud.loadbalancer.cache.ttl=5s
#关闭ribbon，才会启用cloud-loadbalance
spring.cloud.loadbalancer.ribbon.enabled=false
```





## 3.异常拦截返回值



​	默认情况下，gateway报错时会输出简单的错误json，或空白错误页面，但页面中的隐私的信息如错误堆栈会被删掉，调试阶段，可以将它开启，更多选项都在`server.error.`里面

```properties
server.error.whitelabel.enabled=true
server.error.include-exception=true
server.error.include-message=always
```



这里的控制逻辑在`org.springframework.boot.web.reactive.error.DefaultErrorAttributes#getErrorAttributes(org.springframework.web.reactive.function.server.ServerRequest, org.springframework.boot.web.error.ErrorAttributeOptions)`方法里，如果没在`server.error.`配置里开启的数据，都被移除了。

```java
	
	@Override
	public Map<String, Object> getErrorAttributes(ServerRequest request, ErrorAttributeOptions options) {
		Map<String, Object> errorAttributes = getErrorAttributes(request, options.isIncluded(Include.STACK_TRACE));
		if (this.includeException != null) {
			options = options.including(Include.EXCEPTION);
		}
        //remove了
		if (!options.isIncluded(Include.EXCEPTION)) {
			errorAttributes.remove("exception");
		}
        //remove了
		if (!options.isIncluded(Include.STACK_TRACE)) {
			errorAttributes.remove("trace");
		}
		if (!options.isIncluded(Include.MESSAGE) && errorAttributes.get("message") != null) {
			errorAttributes.put("message", "");
		}
        //remove了
		if (!options.isIncluded(Include.BINDING_ERRORS)) {
			errorAttributes.remove("errors");
		}
		return errorAttributes;
	}
```





## 4.断路器

​		如果使用的Resilience4J作为断路器，一定要进行配置，因为默认的断路器超时时间是1秒，这肯定是不够的。自定义一个Customizer，将超时时间设为60秒。

```java
@Configuration(proxyBeanMethods = false)
public class Resilience4JConfig {

	@Bean
	public Customizer<ReactiveResilience4JCircuitBreakerFactory> customReactiveResilience4j() {
		return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
				.circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
				.timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(60)).build())
				.build());
	}

}
```



需要添加的依赖是，才能使用resilience4j。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```







## 5.路由顺序

生成RouteLocator是没有顺序的，不会按照最佳匹配来处理，要想让生成的RouteLocator排在前面可以使用`@Order`注解，添加注解的一定优先于不加注解的，注解里面填的数字越小越优先。

```java
	@Bean
	@Order(-1)
	public RouteLocator test(RouteLocatorBuilder builder) {
		return builder.routes()
				.route(ps -> ps.path("/test")
						.uri("lb://server")
				)
				.build();
	}
```









## 6.添加filter的方式

下面两种添加filter的方式有什么区别吗？

```java
public RouteLocator test(RouteLocatorBuilder builder) {
    return builder.routes()
        .route(ps -> ps.path("/aaa")
               .filters(sp -> sp.filter((ex, chain) -> chain.filter(ex)))
               .uri("lb://server")
              )
        
        .route(ps -> ps.path("/bbb")
               .uri("lb://server")
               .filter((ex, chain) -> chain.filter(ex))
              )
        .build();
}
```

答:

​			一个是在执行path后，使用`filters`添加，一个是在uri方法后使用`filter`方法添加。既然是filter链肯定是要排序的，第一种方式添加filter会被包装成OrderFilter，排序号为0，第二种方式filter不会被包装，没有排序号。

​			gateway默认提供的十个globalFilter是实现Order接口的，所以第一种方式添加的filter，会插入到globalFIlter之间（具体顺序请查看每个globalFilter的顺序），第二种方式加入的filter会插入在所有globalFilter之后。

































