---
layout:     post
title:      "springGateway常见类分析"
subtitle:   ""
date:       2020-09-04
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springGateway
- spring
typora-root-url: ..
---



# springGateway常见类分析



​	本文将`springGateway`的重要类提出来，从宏观角度理解下这个框架。





### 1.HttpHandler

​		底层reactor将netty解析好的request封装成request，response交给此类处理。这是最原始的处理请求的接口，最接近底层netty的。

```java
public interface HttpHandler {
	Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response);
}
```



### 2.WebHandler

​		HttpHandler的另一个抽象，参数exchange包装了request + response。此类类比于j2ee里面的servlet。

```java
public interface WebHandler {

	Mono<Void> handle(ServerWebExchange exchange);

}
```



### 3.WebFilter

​		配合WebHandler使用，提供拦截器功能，此类类比于j2ee里面的filter

```java
public interface WebFilter {

	Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain);

}
```



### 4.WebHandlerDecorator

​		WebHandler的代理类，全部委托给代理类执行。仍然实现WebHandler接口。

```java
public class WebHandlerDecorator implements WebHandler {

	private final WebHandler delegate;
    
	public WebHandlerDecorator(WebHandler delegate) {
		this.delegate = delegate;
	}
	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		return this.delegate.handle(exchange);
	}
}

```



### 5.FilteringWebHandler

​		将WebFilter和WebHandler组合在一起，变成一个WebHandler，执行此WebHandler之前会先经过WebFilter链。

```java
public class FilteringWebHandler extends WebHandlerDecorator {

	private final DefaultWebFilterChain chain;


	public FilteringWebHandler(WebHandler handler, List<WebFilter> filters) {
		super(handler);
		this.chain = new DefaultWebFilterChain(handler, filters);
	}


	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		return this.chain.filter(exchange);
	}

}

```



### 6.WebExceptionHandler

​		异常处理，可以配置多个异常处理，如果已经处理完成则返回Mono<VOid>，否则返回Mono<Exception>，这样轮到下一个WebExceptionHandler处理。

```java
public interface WebExceptionHandler {

	Mono<Void> handle(ServerWebExchange exchange, Throwable ex);
}
```



### 7.ExceptionHandlingWebHandler

​		仍然是WebHandler的代理类，使用try catch包住执行逻辑，抛异常时使用exceptionHandlers处理异常。

```java
public class ExceptionHandlingWebHandler extends WebHandlerDecorator {

	private final List<WebExceptionHandler> exceptionHandlers;


	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		Mono<Void> completion;
		try {
			completion = super.handle(exchange);
		}
		catch (Throwable ex) {
			completion = Mono.error(ex);
		}

		for (WebExceptionHandler handler : this.exceptionHandlers) {
			completion = completion.onErrorResume(ex -> handler.handle(exchange, ex));
		}
		return completion;
	}

}

```



### 8.HttpWebHandlerAdapter

​		同时实现了WebHandler 和 HttpHandler接口，提供适配转换功能，将WebHandler适配成HttpHandler供底层调用，因为netty底层需要的是HttpHandler实现类，我们框架是以WebHandler作为基础开发的。

```java
public class HttpWebHandlerAdapter extends WebHandlerDecorator implements HttpHandler {

	@Override
	public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
		ServerWebExchange exchange = createExchange(request, response);
		return getDelegate().handle(exchange)
				.doOnSuccess(aVoid -> logResponse(exchange))
				.onErrorResume(ex -> handleUnresolvedError(exchange, ex))
				.then(Mono.defer(response::setComplete));
	}

}
```





### 9.DispatcherHandler

​		实现了WebHandler接口，类似于springMvc的DispatcherServlet一样，先是通过handlerMapping查找mapping，在通过handlerAdapter执行mapping，再通过`HandlerResultHandler`处理返回结果。

```java
public class DispatcherHandler implements WebHandler, ApplicationContextAware {
	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		return Flux.fromIterable(this.handlerMappings)
				.concatMap(mapping -> mapping.getHandler(exchange))
				.next()
				.switchIfEmpty(createNotFoundError())
				.flatMap(handler -> invokeHandler(exchange, handler))
				.flatMap(result -> handleResult(exchange, result));
	}
}
```





### 10.GlobalFilter

​		这是gateway独有的filter，签名和WebFilter一样，内置十个GloblFilter用来做路由功能。

一般不需要创建GlobalFilter。

```java
public interface GlobalFilter {

	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);

}
```



### 11.GatewayFilter

​		也是gateway独有的，签名和GlobalFilter一样，为了区分是系统默认的还是用户自定义了，我们写gateway的filter通常实现此接口。

```java
public interface GatewayFilter extends ShortcutConfigurable {

	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);

}
```



### 10.Route

​		gateway独有的类，系类相当于数据类，只是用来将路由相关的类，整理在一起。

- uri 标识目标地址

- predicate 判断请求是否符合此route

- gatewayFilters 存储的filter。

```java
public class Route implements Ordered {

	private final String id;

	private final URI uri;

	private final int order;

	private final AsyncPredicate<ServerWebExchange> predicate;

	private final List<GatewayFilter> gatewayFilters;

```



### 11. RouteLocator

​	gateway独有的类，使用Flux封装多个Route，提供Route的存储功能。

```java

public interface RouteLocator {

	Flux<Route> getRoutes();

}
```



### 12.FilteringWebHandler

​		gateway独有的类，和上面webFlux也有这个类，只是重名了而已。

此类实现了WebHandler接口，但实际执行逻辑是多个GatewayFIlter组成的FIlter链。

​		构造方法里传入globalFilters，handler方法里，从route上获取GatewayFilter，将用户Filter和全局FIlter合并成FilterChain，再执行。

​		此类可以看成在tomcat环境下，只有filter链，没有servlet。

```java
public class FilteringWebHandler implements WebHandler {

	protected static final Log logger = LogFactory.getLog(FilteringWebHandler.class);

	private final List<GatewayFilter> globalFilters;

	public FilteringWebHandler(List<GlobalFilter> globalFilters) {
		this.globalFilters = loadFilters(globalFilters);
	}

	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
		List<GatewayFilter> gatewayFilters = route.getFilters();

		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
		combined.addAll(gatewayFilters);
		// TODO: needed or cached?
		AnnotationAwareOrderComparator.sort(combined);

		if (logger.isDebugEnabled()) {
			logger.debug("Sorted gatewayFilterFactories: " + combined);
		}

		return new DefaultGatewayFilterChain(combined).filter(exchange);
	}
```





### 13.RoutePredicateHandlerMapping

​	gateway独有的，实现了HandlerMapping接口，查找handler时，将查找RouteLocator里面的匹配的Route，将route存入exchange里面，用于构造FilteringWebHandler。







### 14.逻辑图



![springGateway](/img/in/2020-09-04-springGateway常见类分析/springGateway.svg)





## 总结



​		以上就是gateway使用到的主要类，因为gateway是基于springWebFlux实现的，所以流程还是webFlux的流程，gateway对其进行扩充，使用十个globleFilter实现了网关功能。





















