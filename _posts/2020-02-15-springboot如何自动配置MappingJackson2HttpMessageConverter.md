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

springboot开箱即用，写一个controller，再配合RestController，无需其他配置，就能直接返回json到前台，那他是怎么做到的呢，如何定制它，因为只靠默认是不能满足我们的需求的，花了点时间看下源码，进行总结。



## springmvc如何将返回值转成json的

我们知道，当请求到达DispatchServlet时，该servlet会查找到对应的mapper，然后查找与之适配的HandlerAdapter。使用HandlerAdapter.handler(request,response)来处理请求。

下面看HandlerAdapter里的invokeHandlerMethod方法



```java

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandler#invokeHandlerMethod
    
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



下面看invocableMethod真实调用时的源码：

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



而returnValueHandlers是如何处理返回值的呢下面看returnValueHandlers是什么时候创建的。



查看handlerAdapet的源码发现有个这个方法，用于初始化多个returnValueHandlers

```
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



```
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



由上可以看出返回值是由RequestResponseBodyMethodProcessor处理的，而RequestResponseBodyMethodProcessor处理逻辑是使用RequestMapperHandlerAdaper里面的MessageConverters依次尝试处理直到找到一个能处理的处理器为止，这里真正使用的MessageConverter是MappingJackson2HttpMessageConverter，所以能将返回值转成json。





## 看看MappingJackson2HttpMessageConverter什么时候设置进HandlerAdapter里面的

因为SpringBoot的个默认自动配置，我们找到了WebMvcAutoConfiguration这个类，我们在这个类里面找到了这个方法,他将RequestMappingHandlerAdapter创建为Bean。

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



查看`super.requestMappingHandlerAdapter()`，我们找到下面的方法

```java
protected final List<HttpMessageConverter<?>> getMessageConverters() {
		if (this.messageConverters == null) {
			this.messageConverters = new ArrayList<>();
			configureMessageConverters(this.messageConverters);
			if (this.messageConverters.isEmpty()) {
				addDefaultHttpMessageConverters(this.messageConverters);
			}
			extendMessageConverters(this.messageConverters);
		}
		return this.messageConverters;
	}
```



对于configureMessageConverters()方法,EnableWebMvcConfiguration类正好属于DelegatingWebMvcConfiguration的子类，并重写了这个方法，继续进入父类的configureMessageConverters方法，发现其委托给configurers，configurers是通过@Autowired注入进来的，该类正好是WebMvcAutoConfiguration里面的WebMvcAutoConfigurationAdapter内部类，所以找到方法的真实执行逻辑。

```java
WebMvcAutoConfiguration#WebMvcAutoConfigurationAdapter类

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



configureMessageConverters方法通过构造方法传入的`ObjectProvider<HttpMessageConverters> messageConvertersProvider`, 此方法提供了`HttpMessageConverters`。



顺着这个类名，我们找到了`HttpMessageConvertersAutoConfiguration`，他也是自动配置的类，它提供了`HttpMessageConverters`这个Bean

```java
	org.springframework.boot.autoconfigure.http.HttpMessageConverters
	
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
		List<HttpMessageConverter<?>> combined = getCombinedConverters(converters,
				addDefaultConverters ? getDefaultConverters() : Collections.emptyList());
		combined = postProcessConverters(combined);
		this.converters = Collections.unmodifiableList(combined);
	}
```

从上面两段代码看出，`HttpMessageConverters`提供了多个HttpMessageConverter，而创建`HttpMessageConverters`时，一方面从构造方法里面查找到的`ObjectProvider<HttpMessageConverter<?>> convertersProvider`里面查找HttpMessageConverter，另一方面从`getDefaultConverters()`获取默认的HttpMessageConverter，再聚合一个放入`HttpMessageConverters` 提供出去。

下面看看`ObjectProvider<HttpMessageConverter<?>> convertersProvider`里面从哪里来的，

`convertersProvider`是从工厂中获取的，我们发现两个地方创建了MessageConverter。



```java
一个是org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration.StringHttpMessageConverterConfiguration里面

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



```java
另一个是org.springframework.boot.autoconfigure.http.JacksonHttpMessageConvertersConfiguration.MappingJackson2HttpMessageConverterConfiguration#mappingJackson2HttpMessageConverter

	@Bean
		@ConditionalOnMissingBean(value = MappingJackson2HttpMessageConverter.class,
				ignoredType = { "org.springframework.hateoas.mvc.TypeConstrainedMappingJackson2HttpMessageConverter",
						"org.springframework.data.rest.webmvc.alps.AlpsJsonHttpMessageConverter" })
		public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter(ObjectMapper objectMapper) {
			return new MappingJackson2HttpMessageConverter(objectMapper);
		}

```



这就解释了为什么handlerAdapter里面会有`MappingJackson2HttpMessageConverter`了。





## 定制MappingJackson2HttpMessageConverter

上面解释了controller返回值是如何变成json的，messageConverter是何时设置的，但如何定制呢？

`MappingJackson2HttpMessageConverter`处理的核心功能是由`ObjectMapper`提供的，上面方法中`ObjectMapper`是注入进去的，说明ObjectMapper是在bean工厂中，那何时设置到bean工厂中的呢。



我又找到了这个类JacksonAutoConfiguration,也是一个自动配置类

```java
	@Configuration
	@ConditionalOnClass(Jackson2ObjectMapperBuilder.class)
	static class JacksonObjectMapperConfiguration {

		@Bean
		@Primary
		@ConditionalOnMissingBean
		public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
			return builder.createXmlMapper(false).build();
		}

	}


@Bean
		@ConditionalOnMissingBean
		public Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder(
				List<Jackson2ObjectMapperBuilderCustomizer> customizers) {
			Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
			builder.applicationContext(this.applicationContext);
			customize(builder, customizers);
			return builder;
		}

		@Bean
		public StandardJackson2ObjectMapperBuilderCustomizer standardJacksonObjectMapperBuilderCustomizer(
				ApplicationContext applicationContext, JacksonProperties jacksonProperties) {
			return new StandardJackson2ObjectMapperBuilderCustomizer(applicationContext, jacksonProperties);
		}

```



由上可以看出，`ObjectMapper`的创建是使用了`Jackson2ObjectMapperBuilder`,而`Jackson2ObjectMapperBuilder`的创建又被`Jackson2ObjectMapperBuilderCustomizer`定制过，

而`StandardJackson2ObjectMapperBuilderCustomizer`的创建 使用了`JacksonProperties`和`ApplicationContext`。。`JacksonProperties`对应了application.yml中的jackson相关的配置，这就是说我们在application.yml中配置的内容，最终会反应到创建ObjectMapper上。





## 反向总结一下

application.yml中的jackson配置，-> `StandardJackson2ObjectMapperBuilderCustomizer` -> `Jackson2ObjectMapperBuilder` -> `ObjectMapper` 

-> Jakson2HttmMessageConverter ->`HttpMessageConverters` -> List<HttpMessageConverter>

->RequestMappingHandlerMapperAdapter -> RequestResponseBodyMethodProcessor



从而使用RequestResponseBodyMethodProcessor 来处理controller的返回值。前台展示json。



下章将自己定制ObjectMapper



