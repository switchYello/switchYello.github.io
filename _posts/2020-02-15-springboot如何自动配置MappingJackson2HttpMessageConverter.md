---

layout:     post
title:      "springboot如何自动配置MappingJackson2HttpMessageConverter"
subtitle:   ""
date:       2020-02-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- jackson
- HttpMessageConverter
- json
typora-root-url: ..
---



​	

`springboot`开箱即用，写一个`Controller`，再配合`RestController`，无需其他配置，就能直接返回`Json`到前台，那他是怎么做到的呢?，如何定制它?，因为只靠默认是不能满足我们的需求的，花了点时间看下源码，进行如下总结。



## springmvc如何将返回值转成json的

我们知道，当请求到达`DispatchServlet`时，该`servlet`会查找到对应的`HandlerMapper`，然后查找与之适配的`HandlerAdapter`。使用`HandlerAdapter.handler(request,response)`来处理请求。

下面看`HandlerAdapter`里的`invokeHandlerMethod`方法是如何处理请求的



> org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandler#invokeHandlerMethod

```java
    
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        //设置用来解析参数的argumentResolvers到invocableMethod
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        //设置用来处理返回值的returnValueHandlers到invocableMethod
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);

        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            LogFormatUtils.traceDebug(logger, traceOn -> {
                String formatted = LogFormatUtils.formatValue(result, !traceOn);
                return "Resume with async result [" + formatted + "]";
            });
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }
        //真实调用
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }

        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
```



下面看`invocableMethod`真实调用时的源码：

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		//调用我们写的controller，获取返回值
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);

		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				disableContentCachingIfNecessary(webRequest);
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
            //使用returnValueHandlers处理返回值
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(formatErrorForReturnValue(returnValue), ex);
			}
			throw ex;
		}
	}

```



上面可以看到，`HandlerAdapter`会调用我们字节写的`controller`方法，`controller`返回结果使用`returnValueHandlers`来处理的。

下面我看带着疑问，什么是`returnValueHandlers`？他是什么时候被创建的？他是如何处理返回值的？



查看`handlerAdapet`的源码发现有个这个方法，用于初始化多个`returnValueHandlers`

```java
	//此方法会当属性设置完毕后被spring回掉
	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		initControllerAdviceCache();

		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		//这个方法初始化ReturnValueHandlers
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
```



```java
	//可以看到此方法初始化了多个HandlerMethodReturnValueHandler
	private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>();

		// Single-purpose return value types
		handlers.add(new ModelAndViewMethodReturnValueHandler());
		handlers.add(new ModelMethodProcessor());
		handlers.add(new ViewMethodReturnValueHandler());
		handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters(),
				this.reactiveAdapterRegistry, this.taskExecutor, this.contentNegotiationManager));
		handlers.add(new StreamingResponseBodyReturnValueHandler());
		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));
		handlers.add(new HttpHeadersReturnValueHandler());
		handlers.add(new CallableMethodReturnValueHandler());
		handlers.add(new DeferredResultMethodReturnValueHandler());
		handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));

		// Annotation-based return value types
		handlers.add(new ModelAttributeMethodProcessor(false));
		
		//转json真正使用的handlers，参数传入了adapter内的所有MessageConverters
		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));

		// Multi-purpose return value types
		handlers.add(new ViewNameMethodReturnValueHandler());
		handlers.add(new MapMethodProcessor());

		// Custom return value types
		if (getCustomReturnValueHandlers() != null) {
			handlers.addAll(getCustomReturnValueHandlers());
		}

		// Catch-all
		if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
			handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
		}
		else {
			handlers.add(new ModelAttributeMethodProcessor(true));
		}

		return handlers;
	}
```



看上面的源码，回答下上面的疑问

什么是`returnValueHandlers`?

他是用于处理`controller`的返回值的处理器，它都多个实现类存在于`MapperAdapter`里，不同的`returnValueHandlers`能处理不同类型的返回值，责任链模式调用。



他是什么时候被创建的？

他在`afterPropertiesSet`方法里被初始化，这个方法是`InitializingBean`接口的方法，从名字可以看出这个方法在类属性设置完成后被`Spring`回掉，所以在这里创建了多个`returnValueHandlers`存在在list里，我们要注意的是，其中有一个`RequestResponseBodyMethodProcessor`，创建它时将`HandlerAdapter`里面的所有`MessageConverter`都传进去了。



他是如何处理返回值的？

他是使用构造参数传入的`MessageConverter`处理的，其中有一个`MessageConverter`就是`MappingJackson2HttpMessageConverter`，这个类很熟悉，就是他将返回值转成`Json`的。



## 看看MappingJackson2HttpMessageConverter什么时候设置进HandlerAdapter里面的

既然知道返回值是何时被转成`Json`的，又有了新疑问，那就是`HandlerAdapter`里面的`MessageConverter`是什么时候被设置的？，以及`MappingJackson2HttpMessageConverter`是什么时候被设置的？



我们找到了`WebMvcAutoConfiguration`这个类，这里我们看到了自动初始化`RequestMappingHandlerAdapter`的方法，进入方法内部查看，我们找到了设置`HttpMessageConverter`的过程。

> 创建RequestMappingHandlerAdapter的方法

```java
@Bean
@Override
public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
			RequestMappingHandlerAdapter adapter = super.requestMappingHandlerAdapter();
			adapter.setIgnoreDefaultModelOnRedirect(
					this.mvcProperties == null || this.mvcProperties.isIgnoreDefaultModelOnRedirect());
			return adapter;
		}
```

> 查看`super.requestMappingHandlerAdapter()`，我们找到下面的方法

```java
@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
    RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
    adapter.setContentNegotiationManager(mvcContentNegotiationManager());
    //创建后，设置了MessageConverters
    adapter.setMessageConverters(getMessageConverters());
    adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer());
    adapter.setCustomArgumentResolvers(getArgumentResolvers());
    adapter.setCustomReturnValueHandlers(getReturnValueHandlers());
   	    .
        .
        .
```



```java
protected final List<HttpMessageConverter<?>> getMessageConverters() {
		if (this.messageConverters == null) {
			this.messageConverters = new ArrayList<>();
            //配置MessageConverters的方法
			configureMessageConverters(this.messageConverters);
			if (this.messageConverters.isEmpty()) {
				addDefaultHttpMessageConverters(this.messageConverters);
			}
			extendMessageConverters(this.messageConverters);
		}
		return this.messageConverters;
	}
```



`configureMessageConverters(this.messageConverters)`方法被父类重写了

> org.springframework.web.servlet.config.annotation.DelegatingWebMvcConfiguration#configureMessageConverters

```java
	@Override
	protected void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
		this.configurers.configureMessageConverters(converters);
	}
```



上面方法中啥是`configurers`？，什么时候被创建的？

我们发现`configurers`是通过创建`DelegatingWebMvcConfiguration`时，通过构造方法注入的`List<WebMvcConfigurer> configurers`创建的。



我们在`WebMvcAutoConfiguration`里面发现`WebMvcAutoConfigurationAdapter`类继承了`WebMvcConfigurer`，且会被自动注入到Spring内的，所以删改你创建`DelegatingWebMvcConfiguration`时使用的`WebMvcConfigurer`就是此类，此类的`configureMessageConverters`方法会对`HttpMessageConverter`进行配置。看一下它的源码。



> WebMvcAutoConfiguration#WebMvcAutoConfigurationAdapter类

```java


public WebMvcAutoConfigurationAdapter(ResourceProperties resourceProperties, WebMvcProperties mvcProperties,
				ListableBeanFactory beanFactory, ObjectProvider<HttpMessageConverters> messageConvertersProvider,
				ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider) {
			this.resourceProperties = resourceProperties;
			this.mvcProperties = mvcProperties;
			this.beanFactory = beanFactory;
			this.messageConvertersProvider = messageConvertersProvider;
			this.resourceHandlerRegistrationCustomizer = resourceHandlerRegistrationCustomizerProvider.getIfAvailable();
		}


@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
			this.messageConvertersProvider.ifAvailable((customConverters) -> converters.addAll(customConverters.getConverters()));
		}
```



`configureMessageConverters`方法通过`messageConvertersProvider`来设置`MessageConverter`到list中，`messageConvertersProvider`是通过构造方法注入进来的，我们要找找`messageConvertersProvider`是什么时候注入Spring的？



顺着泛型信息，我们找到了`HttpMessageConvertersAutoConfiguration`，他也是自动配置的类，它提供了`HttpMessageConverters`这个Bean。

> org.springframework.boot.autoconfigure.http.HttpMessageConverters

```java
	
   private final List<HttpMessageConverter<?>> converters;

public HttpMessageConvertersAutoConfiguration(ObjectProvider<HttpMessageConverter<?>> convertersProvider) {
		this.converters = convertersProvider.orderedStream().collect(Collectors.toList());
	}
        
    @Bean
	@ConditionalOnMissingBean
	public HttpMessageConverters messageConverters() {
		return new HttpMessageConverters(this.converters);
	}
```

而`HttpMessageConverters`的构造方法里面是这样处理的

```java
	public HttpMessageConverters(boolean addDefaultConverters, Collection<HttpMessageConverter<?>> converters) {
        //注意这里的getDefaultConverters()方法
		List<HttpMessageConverter<?>> combined = getCombinedConverters(converters,                                                                       addDefaultConverters ? getDefaultConverters() : Collections.emptyList());
		combined = postProcessConverters(combined);
		this.converters = Collections.unmodifiableList(combined);
	}
```

从`HttpMessageConvertersAutoConfiguration`类里面可以看出，他本身会在构造方法里通过`getDefaultConverters()`方法创建默认的`MessageConverter`,其次他还会通过`HttpMessageConvertersAutoConfiguration`的构造方法注入外界的`HttpMessageConverter`，合并在一起。



下面看看构造方法里的`ObjectProvider<HttpMessageConverter<?>> convertersProvider`里面从哪里来的？

我们发现两个地方创建了`MessageConverter`。

> 一个是org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration.StringHttpMessageConverterConfiguration里面创建的`StringHttpMessageConverter`

```java

	@Configuration
	@ConditionalOnClass(StringHttpMessageConverter.class)
	@EnableConfigurationProperties(HttpProperties.class)
	protected static class StringHttpMessageConverterConfiguration {

		private final HttpProperties.Encoding properties;

		protected StringHttpMessageConverterConfiguration(HttpProperties httpProperties) {
			this.properties = httpProperties.getEncoding();
		}

		@Bean
		@ConditionalOnMissingBean
		public StringHttpMessageConverter stringHttpMessageConverter() {
			StringHttpMessageConverter converter = new StringHttpMessageConverter(this.properties.getCharset());
			converter.setWriteAcceptCharset(false);
			return converter;
		}

	}

```

> 另一个是org.springframework.boot.autoconfigure.http.JacksonHttpMessageConvertersConfiguration.MappingJackson2HttpMessageConverterConfiguration创建的`MappingJackson2HttpMessageConverter`

```java

	@Bean
		@ConditionalOnMissingBean(value = MappingJackson2HttpMessageConverter.class,
				ignoredType = { "org.springframework.hateoas.mvc.TypeConstrainedMappingJackson2HttpMessageConverter",
						"org.springframework.data.rest.webmvc.alps.AlpsJsonHttpMessageConverter" })
		public  mappingJackson2HttpMessageConverter(ObjectMapper objectMapper) {
			return new MappingJackson2HttpMessageConverter(objectMapper);
		}

```



这就解释了为什么`HandlerAdapter`里面会有`MappingJackson2HttpMessageConverter`了。

原来是创建`HandlerAdapter`时的`getHandlerMessages`方法做了这些工作。



## 定制MappingJackson2HttpMessageConverter

上面解释了`HandlerAdapter`里面的`HandlerMessage`是怎么来的。

但又有一个疑问，问什么我们在`Springboot`的配置文件里能配置`JacksonMessageConverter`工作的？



我们知道`MappingJackson2HttpMessageConverter`处理的核心功能是由`ObjectMapper`提供的，所有配置都在`ObjectMapper`里，所以创建`MappingJackson2HttpMessageConverter`时传入的`ObjectMapper`是经过`Springboot`创建并根据配置文件配置过的。

疑问是`ObjectMapper`是什么时候创建的？如何被配置的？



我又找到了这个类`JacksonAutoConfiguration`,也是一个自动配置类

```java
	@Configuration
	@ConditionalOnClass(Jackson2ObjectMapperBuilder.class)
	static class JacksonObjectMapperConfiguration {

       //通过Jackson2ObjectMapperBuilder创建ObjectMapper
        @Bean
        @Primary
        @ConditionalOnMissingBean
        public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
			return builder.createXmlMapper(false).build();
		}

	}

		//通过Jackson2ObjectMapperBuilderCustomizer创建Jackson2ObjectMapperBuilder
		@Bean
		@ConditionalOnMissingBean
		public Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder(
				List<Jackson2ObjectMapperBuilderCustomizer> customizers) {
			Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
			builder.applicationContext(this.applicationContext);
			customize(builder, customizers);
			return builder;
		}
		/*
		创建StandardJackson2ObjectMapperBuilderCustomizer的过程，使用了JacksonProperties
		而JacksonProperties就对应了Springboot的配置文件里面的“spring.jackson”开始的配置
		*/
		@Bean
		public StandardJackson2ObjectMapperBuilderCustomizer standardJacksonObjectMapperBuilderCustomizer(
				ApplicationContext applicationContext, JacksonProperties jacksonProperties) {
			return new StandardJackson2ObjectMapperBuilderCustomizer(applicationContext, jacksonProperties);
		}

```



上面`ObjectMapper`的创建过程也就能看出为什么`application.yml`里的配置能影响到`ObjectMapper`的行为了。



## 反向总结一下

`application.yml`中的`jackson`配置，

导致`JacksonProperties`里面的属性变化，

导致`JacksonAutoConfiguration`里面的`StandardJackson2ObjectMapperBuilderCustomizer`变化，

导致`JacksonAutoConfiguration`里面的`Jackson2ObjectMapperBuilder`变化，

导致创建的`JacksonAutoConfiguration`里面的`ObjectMapper`属性变化，然后`ObjectMapper`被注入`Spring`

然后`JacksonHttpMessageConvertersConfiguration`里面创建`MappingJackson2HttpMessageConverter` 时从Spring获取`ObjectMapper`。



`HttpMessageConvertersAutoConfiguration`里面创建`HttpMessageConverters`时从`Spring`获取`Jakson2HttmMessageConverter` 。



`WebMvcAutoConfiguration`创建`RequestMappingHandlerMapperAdapter` 时从容器获取了`HttpMessageConverters`，然后设置到`RequestResponseBodyMethodProcessor`里面。



然后`RequestResponseBodyMethodProcessor`使用`Jakson2HttmMessageConverter` 处理`Controller`的返回值，也就实现了根据配置文件来定制返回J`Json`的行为了。



## 完

但大家一定也遇到入：

​	配置文件里配置列`Jackson`但不生效？

​	自己手动创建`ObjectMapper`来代替`Springboot`帮我们创建的，但不生效？

​	如何做才是最佳实践？

下篇讲讲上面三个问题。

