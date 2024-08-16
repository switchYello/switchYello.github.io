---

layout:     post
title:      "SpringBoot访问静态文件的原理总结"
subtitle:   ""
date:       2020-04-12
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- java
- springboot
- defaultServlet
- springboot 静态文件
typora-root-url: ..
---



​        上一篇写了使用[SpringBoot访问静态文件的几种方法](https://www.huangchaoyu.com/2020/04/10/SpringBoot%E8%AE%BF%E9%97%AE%E9%9D%99%E6%80%81%E6%96%87%E4%BB%B6%E7%9A%84%E5%87%A0%E7%A7%8D%E6%96%B9%E6%B3%95/) ,这篇文章来讲一讲配置Springboot访问静态文件的原理，为何简单配置就能实现静态文件的加载。



* TOC
{:toc}
 ## 我们的做法

​        我们是重写了`WebMvcConfigurationSupport`的`addResourceHandlers`方法，在`registry`内配置映射关系来实现静态文件映射的。

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {
    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        //访问文件夹写法，注意文件夹路径要以file开头，以 / 结尾
        registry.addResourceHandler("/static/**", "/download/**").addResourceLocations("file:F:/娱乐/","classpath:/static/");
    }
}
```

​       

​        追踪`addResourceHandlers`的父类调用关系。是一个加了`@Bean`注解的方法调用了此方法。

​        此方法会使用我们配置的`registry`创建一个`HandlerMapping`处理，如果没配置则不会构造这个`HandlerMapping`。

```java
	@Bean
	@Nullable
	public HandlerMapping resourceHandlerMapping(
			@Qualifier("mvcUrlPathHelper") UrlPathHelper urlPathHelper,
			@Qualifier("mvcPathMatcher") PathMatcher pathMatcher,
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

		Assert.state(this.applicationContext != null, "No ApplicationContext set");
		Assert.state(this.servletContext != null, "No ServletContext set");

        //这个registry作参数
		ResourceHandlerRegistry registry = new ResourceHandlerRegistry(this.applicationContext,
				this.servletContext, contentNegotiationManager, urlPathHelper);
		//回掉我们重写的方法
        addResourceHandlers(registry);
        //如果我们没重写，直接返回null
		AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();
		if (handlerMapping == null) {
			return null;
		}
        //如果重写了，能获取到HandlerMapping返回
		handlerMapping.setPathMatcher(pathMatcher);
		handlerMapping.setUrlPathHelper(urlPathHelper);
		handlerMapping.setInterceptors(getInterceptors(conversionService, resourceUrlProvider));
		handlerMapping.setCorsConfigurations(getCorsConfigurations());
		return handlerMapping;
	}
```



​        构造过程，查看上面代码的`registry.getHandlerMapping()`调用。

​        他会根据我们的配置构造`SimpleUrlHandlerMapping`，这个`SimpleUrlHandlerMapping`内部保存多个`HttpRequestHandler`，将每个请求路径映射成一个`HttpRequestHandler`。

```java
	protected AbstractHandlerMapping getHandlerMapping() {
		if (this.registrations.isEmpty()) {
			return null;
		}
		//根据registrations，构造多个HttpRequestHandler存入urlMap里面
		Map<String, HttpRequestHandler> urlMap = new LinkedHashMap<>();
		for (ResourceHandlerRegistration registration : this.registrations) {
			for (String pathPattern : registration.getPathPatterns()) {
				ResourceHttpRequestHandler handler = registration.getRequestHandler();
				if (this.pathHelper != null) {
					handler.setUrlPathHelper(this.pathHelper);
				}
				if (this.contentNegotiationManager != null) {
					handler.setContentNegotiationManager(this.contentNegotiationManager);
				}
				handler.setServletContext(this.servletContext);
				handler.setApplicationContext(this.applicationContext);
				try {
					handler.afterPropertiesSet();
				}
				catch (Throwable ex) {
					throw new BeanInitializationException("Failed to init ResourceHttpRequestHandler", ex);
				}
				urlMap.put(pathPattern, handler);
			}
		}
        //根据urlMap里面的配置，构造SimpleUrlHandlerMapping
		return new SimpleUrlHandlerMapping(urlMap, this.order);
	}
```

​        

​        上面的配置将断点打到这里，查看`urlMap`里面的值,他的`key`是拦截的路径，`value`是构造的`ResourceHttpRequestHandler`，这个`Handler`里面配置的是两个路径分别是`/static/**`和`/download/**`。

​        值都是ResourceHttpRequestHandler实例，里面保存两个路径是`file:F:/娱乐/`，`classpath:/static/`。这样浏览器发送的请求如何匹配`/static/**`就会被下面的`handler`处理，去配置的两个存储位置查找资源。

![image-20200412144041449](/img/in/2020-04-12-SpringBoot%E8%AE%BF%E9%97%AE%E9%9D%99%E6%80%81%E6%96%87%E4%BB%B6%E7%9A%84%E5%8E%9F%E7%90%86%E6%80%BB%E7%BB%93/image-20200412144041449.png)



## 何时调用HandlerMapping的

​        将断点打在`Dispathervlet`的`doDispatch`方法上，访问 http://localhost:8080/static/2.jpg，程序进入`getHandler()`方法里。

```java
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
            //遍历handlerMappings列表，找到合适的HandlerMapping
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

​        

​        此方法内遍历的`handlerMapping`如下，他会逐个调用他们的`getHandler()`方法，这样就能找到对应的`ResourceHttpRequestHandler`处理请求了。

![image-20200412145156338](/img/in/2020-04-12-SpringBoot%E8%AE%BF%E9%97%AE%E9%9D%99%E6%80%81%E6%96%87%E4%BB%B6%E7%9A%84%E5%8E%9F%E7%90%86%E6%80%BB%E7%BB%93/image-20200412145156338.png)



## ResourceHttpRequestHandler如何处理请求的



​        查看`ResourceHttpRequestHandler`的`handleRequest`方法。它调用了`getResource(request)`方法获取资源。

```java
@Override
public void handleRequest(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {

   // 获取资源
   Resource resource = getResource(request);
   if (resource == null) {
      logger.debug("Resource not found");
      response.sendError(HttpServletResponse.SC_NOT_FOUND);
      return;
   }
   ..... 省略
```



`getResource`方法使用了责任模式，将多个`location`封装到`resolverChain`中。

```java
	@Nullable
	protected Resource getResource(HttpServletRequest request) throws IOException {
		String path = (String) request.getAttribute(HandlerMapping.PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE);
		path = processPath(path);

		Assert.notNull(this.resolverChain, "ResourceResolverChain not initialized.");
		Assert.notNull(this.transformerChain, "ResourceTransformerChain not initialized.");
		//resolverChain就是责任链模式，将多个locations封装成责任链
		Resource resource = this.resolverChain.resolveResource(request, path, getLocations());
		if (resource != null) {
			resource = this.transformerChain.transform(request, resource);
		}
		return resource;
	}
```



## location的分类

`org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry#getHandlerMapping`方法内调用了`handler.afterPropertiesSet() `方法，再调用`resolveResourceLocations()`方法，此方法将配置的多个路径解析成不同的Location。

```java
Resource resource = applicationContext.getResource(location);
```



请看下面代码，`location`有四种分别是

- `ClassPathResource`   从classpath获取资源
- `FileUrlResource`       从文件获取资源
- `UrlResource`              从url获取资源
- `ClassPathContextResource`     委托给context获取资源



​      下面是将字符串解析成不同`Location`的方法。

```java
@Override
public Resource getResource(String location) {
   Assert.notNull(location, "Location must not be null");

   for (ProtocolResolver protocolResolver : getProtocolResolvers()) {
      Resource resource = protocolResolver.resolve(location, this);
      if (resource != null) {
         return resource;
      }
   }

   if (location.startsWith("/")) {
      return getResourceByPath(location);
   }
   //以classpath开头的，创建ClassPathResource
   else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
      return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
   }
   else {
      try {
         //符合URL规范的返回FileUrlResource或者UrlResource
         // Try to parse the location as a URL...
         URL url = new URL(location);
         return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
      }
      catch (MalformedURLException ex) {
         // 否则返回 ClassPathContextResource
         return getResourceByPath(location);
      }
   }
}
```



## 总结

`Springboot`将我们配置的映射:

```java
 registry.addResourceHandler("/static/**", "/download/**").addResourceLocations("file:F:/娱乐/","classpath:/static/");
```

​        将`addResourceLocations`根据前缀解析成上面四种`Location`对象，在加上配置的路径封装成`ResourceHttpRequestHandler`用于处理请求，多个`ResourceHttpRequestHandler`封装成`SimpleUrlHandlerMapping`对象。

​        `DispatchServlet`内能查找到`SimpleUrlHandlerMapping`内的`ResourceHttpRequestHandler`处理请求。

`ResourceHttpRequestHandler`将内部的`Location`组成责任链，按顺序查找资源。

​       以上就是`SpringBoot`访问静态文件的原理。

