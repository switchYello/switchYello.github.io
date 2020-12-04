---
layout:     post
title:      "springCloud-openFeign使用原理"
subtitle:   ""
date:       2020-08-11
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springcloud
- openfeign
typora-root-url: ..
---



# springCloud-openFeign使用原理



​		本文讲解`spring-cloud`环境下的`openFeign`的用法，探究`spring-cloud`是如何让`openfeign`开箱即用的。本文会假设读者已经熟练使用`openfeign`，对`openFeign`源码已有了解。



* TOC
{:toc}



## 1.先让项目运行起来

新建`spring-cloud`项目，添加如下依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```



下面以获取百度贴吧首页为例。

写一个接口，其中`@FeignClient`注解上`url属性`标识请求的域名，`@GetMapping`标注请求`method` 和`path`，`@RequestParam`标注请求参数。

```java
//准备获取这个网页 https://tieba.baidu.com/f?kw=java

@FeignClient(name = "javaBaCLient", url = "https://tieba.baidu.com")
public interface Baidu {

	@GetMapping(value = "/f")
	String getMainPage(@RequestParam("kw") String kw);

}
```



**启动类上添加注解`@EnableFeignClients`** 。

​		将`Badu.class`类注入到其某个类中，并调用其中的方法，可以看到`spring-cloud`已经自动将上面的接口生成了代理类，一个开箱即用的案例就写好了。下面开始探究下它的实现原理和默认配置。

```java
@Autowired
private Baidu baidu;

@PostConstruct
public void test() {
    String java = baidu.getMainPage("java");
    System.out.println(java);
}
```

### 总结

​			开箱即用，非常简单的就能将Demo跑起来。如果在已经存在的`SpringBoot`项目上添加`spring-cloud-openfeign`，要注意`spring-cloud`的版本必须匹配。这里有一个网址是`spring-cloud`项目的版本对应关系

https://start.spring.io/actuator/info





## 2.自动装配

​	这里可能你会很好去，`spring-cloud`是怎么做的，他到底为我们做了什么？



首先从`@EnableFeignClients`注解上找到`org.springframework.cloud.openfeign.FeignClientsRegistrar`类

查看它的`registerBeanDefinitions`方法，它做了两件事情。

1. 找到`@EnableFeignClients`注解，将其属性`defaultConfiguration`对应的类注册为属性。
2. 扫描带有`@FeignClien`注解的类，将其注册为`FeignClientFactoryBean`

```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
                                    BeanDefinitionRegistry registry) {

    registerDefaultConfiguration(metadata, registry);
    registerFeignClients(metadata, registry);
}
```



这里关键在`FeignClientFactoryBean`类，这是一个`工厂类`，它的`getObject()`方法里面是真正生成Feign接口代理类的。细节请看注释。

```java
@Override
public Object getObject() throws Exception {
    return getTarget();
}


<T> T getTarget() {
    /*
    	1.获取FeignContext，相当于是ApplicationContext容器的子容器。
    		有点类似于Springmvc的那个子容器一样
    	*/
    FeignContext context = this.applicationContext.getBean(FeignContext.class);

    /*
    	2. 从容器里获取Builder，获取的build是经过配置的，下面有详细介绍此方法

    	*/
    Feign.Builder builder = feign(context);

    /*
    	3.这个url就是@FeignClient注解上配置的url
    		这里有意思的一点就是如果没有配置url只配置了name，那么他就会把name属性当成负载均衡的实例名称，这里底层要使用ribbon来做负载均衡，他就会尝试查找此name对应的实例。
    		如果url不为空，如url配置的'http://www.baidu.com' 那么他就会以这个值为服务器地址。

    	*/
    if (!StringUtils.hasText(this.url)) {
        if (!this.name.startsWith("http")) {
            this.url = "http://" + this.name;
        }
        else {
            this.url = this.name;
        }
        this.url += cleanPath();
        //url为空，使用负载均衡方式创建Feign
        return (T) loadBalance(builder, context,
                               new HardCodedTarget<>(this.type, this.name, this.url));
    }
    if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
        this.url = "http://" + this.url;
    }
    String url = this.url + cleanPath();

    /*
    	4. 从容器里获取地城feign.Client，默认就是java原生的那个

    	*/
    Client client = getOptional(context, Client.class);
    if (client != null) {
        if (client instanceof LoadBalancerFeignClient) {
            // not load balancing because we have a url,
            // but ribbon is on the classpath, so unwrap
            client = ((LoadBalancerFeignClient) client).getDelegate();
        }
        builder.client(client);
    }
    /*
    	5.获取targeter，在targeter里面对实例进行配置

    	*/
    Targeter targeter = get(context, Targeter.class);
    return (T) targeter.target(this, builder, context,
                               new HardCodedTarget<>(this.type, this.name, url));
}



/*
	获取Feign.Builder的过程，其实就是从容器里取各种各样的东西，配置进去
	熟悉Feign的同学应该能看出来，分别是logger，encoder，decoder，contract
*/
	protected Feign.Builder feign(FeignContext context) {
		FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
		Logger logger = loggerFactory.create(this.type);

		Feign.Builder builder = get(context, Feign.Builder.class)
				// required values
				.logger(logger)
				.encoder(get(context, Encoder.class))
				.decoder(get(context, Decoder.class))
				.contract(get(context, Contract.class));

		//这里面也有一些配置，是根据配置文件或者注解上指定的配置类来配置的
        //本文不讲这些
		configureFeign(context, builder);

		return builder;
	}
```



### 总结

​		通过`@EnableFeignClients`来引入一个`ImportBeanDefinitionRegistrar`。这个类里面注册了默认配置，扫描了所有的`@FeignClient`接口，将其注册为`FactoryBean`，真实使用时将会调用这个`FactoryBean`的`getObject`方法，此时才会产生代理类，并且生成代理类需要的各种组件都是从容器里获取的。





## 3.默认配置

根据`@EnableFeignClient`注解上的注释，找到了下面这个类`org.springframework.cloud.openfeign.FeignClientsConfiguration`，这是一个配置类，会被自动扫描到，它里面注入了以下组件。



```java

//默认Decoder->SpringDecoder
@Bean
@ConditionalOnMissingBean
public Decoder feignDecoder() {
    return new OptionalDecoder(
        new ResponseEntityDecoder(new SpringDecoder(this.messageConverters)));
}

//默认Encoder->SpringEncoder
@Bean
@ConditionalOnMissingBean
@ConditionalOnMissingClass("org.springframework.data.domain.Pageable")
public Encoder feignEncoder() {
    return new SpringEncoder(this.messageConverters);
}

//如果存在Pageable则用PageableSpringEncoder包装SpringEncoder
@Bean
@ConditionalOnClass(name = "org.springframework.data.domain.Pageable")
@ConditionalOnMissingBean
public Encoder feignEncoderPageable() {
    return new PageableSpringEncoder(new SpringEncoder(this.messageConverters));
}

//默认Contract->SpringMvcContract
//在这里spring-cloud重用了springmvc里面的注解，抛弃了openfeign原生注解，可以看下此类的源码
@Bean
@ConditionalOnMissingBean
public Contract feignContract(ConversionService feignConversionService) {
    return new SpringMvcContract(this.parameterProcessors, feignConversionService);
}


//默认重试策略为->NEVER_RETRY。因为一般要组合断路器使用，或者底层ribbon的重试，不需要Feign的重试
@Bean
@ConditionalOnMissingBean
public Retryer feignRetryer() {
    return Retryer.NEVER_RETRY;
}

//默认Builder,关闭重试
@Bean
@Scope("prototype")
@ConditionalOnMissingBean
public Feign.Builder feignBuilder(Retryer retryer) {
    return Feign.builder().retryer(retryer);
}

//默认的logger，没有配置的话默认是Slf4jLogger
@Bean
@ConditionalOnMissingBean(FeignLoggerFactory.class)
public FeignLoggerFactory feignLoggerFactory() {
    return new DefaultFeignLoggerFactory(this.logger);
}

//如果启用hystrix，则会使用HystrixFeign
@Configuration
@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
protected static class HystrixFeignConfiguration {

    @Bean
    @Scope("prototype")
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "feign.hystrix.enabled")
    public Feign.Builder feignHystrixBuilder() {
        return HystrixFeign.builder();
    }

}

```



### 总结

上面就是`spring-cloud`对`Feign`的默认配置了，他们都是以bean的形式放在容器里，我们可以自己提供上面任何组件取而代之，这样我们替换其中任何组件都是非常简单的。





## 4.默认客户端

​	Feign默认使用的是Java原生的客户端，我们实际使用时一般要用`Apache Httpclient` 或者`okhttp`代替，`spring-cloud`为我们默认提供了这两种框架的自动配置，且是自动选择的。

​	配置代码在`org.springframework.cloud.openfeign.FeignAutoConfiguration`里面，看名字就知道这是一个自动装配的配置类。



如果存在`HttpClient` 和 `feign.ApacheHttpClient`，则`HttpClient`就会生效

```java
@Configuration
@ConditionalOnClass(ApacheHttpClient.class)
@ConditionalOnMissingClass("com.netflix.loadbalancer.ILoadBalancer")
@ConditionalOnMissingBean(CloseableHttpClient.class)
@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)
protected static class HttpClientFeignConfiguration {

    private final Timer connectionManagerTimer = new Timer(
        "FeignApacheHttpClientConfiguration.connectionManagerTimer", true);

    @Autowired(required = false)
    private RegistryBuilder registryBuilder;

    private CloseableHttpClient httpClient;
	
    //1.创建HttpClientConnectionManager
    @Bean
    @ConditionalOnMissingBean(HttpClientConnectionManager.class)
    public HttpClientConnectionManager connectionManager(
        ApacheHttpClientConnectionManagerFactory connectionManagerFactory,
        FeignHttpClientProperties httpClientProperties) {
        
        //这里的连接池最大连接数200
        //每路由最大连接数50
        //timeToLive默认900s
        final HttpClientConnectionManager connectionManager = connectionManagerFactory
            .newConnectionManager(httpClientProperties.isDisableSslValidation(),
                                  httpClientProperties.getMaxConnections(),
                                  httpClientProperties.getMaxConnectionsPerRoute(),
                                  httpClientProperties.getTimeToLive(),
                                  httpClientProperties.getTimeToLiveUnit(),
                                  this.registryBuilder);
        
        //这里使用Timer类定时关闭连接池内的空闲连接，默认3s一次
        this.connectionManagerTimer.schedule(new TimerTask() {
            @Override
            public void run() {
                connectionManager.closeExpiredConnections();
            }
        }, 30000, httpClientProperties.getConnectionTimerRepeat());
        return connectionManager;
    }

    
    //2. 创建HttpClient
    @Bean
    public CloseableHttpClient httpClient(ApacheHttpClientFactory httpClientFactory,
                                          HttpClientConnectionManager httpClientConnectionManager,
                                          FeignHttpClientProperties httpClientProperties) {
        RequestConfig defaultRequestConfig = RequestConfig.custom()
            .setConnectTimeout(httpClientProperties.getConnectionTimeout())
            .setRedirectsEnabled(httpClientProperties.isFollowRedirects())
            .build();
        this.httpClient = httpClientFactory.createBuilder()
            .setConnectionManager(httpClientConnectionManager)
            .setDefaultRequestConfig(defaultRequestConfig).build();
        return this.httpClient;
    }

    //3. 创建feign的Client
    @Bean
    @ConditionalOnMissingBean(Client.class)
    public Client feignClient(HttpClient httpClient) {
        return new ApacheHttpClient(httpClient);
    }

    @PreDestroy
    public void destroy() throws Exception {
        this.connectionManagerTimer.cancel();
        if (this.httpClient != null) {
            this.httpClient.close();
        }
    }

}
```



如果存在`okhttp`，则会使用下面的配置,这里就不做解释了。

```java
@Configuration
	@ConditionalOnClass(OkHttpClient.class)
	@ConditionalOnMissingClass("com.netflix.loadbalancer.ILoadBalancer")
	@ConditionalOnMissingBean(okhttp3.OkHttpClient.class)
	@ConditionalOnProperty("feign.okhttp.enabled")
	protected static class OkHttpFeignConfiguration {

		private okhttp3.OkHttpClient okHttpClient;

		@Bean
		@ConditionalOnMissingBean(ConnectionPool.class)
		public ConnectionPool httpClientConnectionPool(
				FeignHttpClientProperties httpClientProperties,
				OkHttpClientConnectionPoolFactory connectionPoolFactory) {
			Integer maxTotalConnections = httpClientProperties.getMaxConnections();
			Long timeToLive = httpClientProperties.getTimeToLive();
			TimeUnit ttlUnit = httpClientProperties.getTimeToLiveUnit();
			return connectionPoolFactory.create(maxTotalConnections, timeToLive, ttlUnit);
		}

		@Bean
		public okhttp3.OkHttpClient client(OkHttpClientFactory httpClientFactory,
				ConnectionPool connectionPool,
				FeignHttpClientProperties httpClientProperties) {
			Boolean followRedirects = httpClientProperties.isFollowRedirects();
			Integer connectTimeout = httpClientProperties.getConnectionTimeout();
			Boolean disableSslValidation = httpClientProperties.isDisableSslValidation();
			this.okHttpClient = httpClientFactory.createBuilder(disableSslValidation)
					.connectTimeout(connectTimeout, TimeUnit.MILLISECONDS)
					.followRedirects(followRedirects).connectionPool(connectionPool)
					.build();
			return this.okHttpClient;
		}

		@PreDestroy
		public void destroy() {
			if (this.okHttpClient != null) {
				this.okHttpClient.dispatcher().executorService().shutdown();
				this.okHttpClient.connectionPool().evictAll();
			}
		}

		@Bean
		@ConditionalOnMissingBean(Client.class)
		public Client feignClient(okhttp3.OkHttpClient client) {
			return new OkHttpClient(client);
		}

	}

```

​	

需要添加如下依赖，`Apache Httpclient`才会自动配置，注意要和Feign版本一致。

```xml
        <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-httpclient</artifactId>
            <version>10.4.0</version>
        </dependency>
```







### 总结

​		`spring-cloud`底层是可以根据项目存在的依赖选择使用`apache httpclient` 或者 `okhttp`来作为底层实现，而且**默认配置都比较合理**，几乎不用改动。

​		需要注意的一点是，项目内只是存在`Apache Httpclient`的依赖是不够的，还需要添加`feign-httpclient`的依赖做适配包才可以。`okhttp`也是同理，都需要一个适配包。





## 5.其他配置

​		还有一个配置类也会被自动扫描到，上面缺少的client相关的bean会从这里获取`org.springframework.cloud.commons.httpclient.HttpClientConfiguration`，以`Apache Httpclient`为例。他的源码是这样的，他的方法都带有`@ConditionalOnMissingBean`注解，所以优先级最低。

```java
	@Configuration
	@ConditionalOnProperty(name = "spring.cloud.httpclientfactories.apache.enabled", matchIfMissing = true)
	@ConditionalOnClass(HttpClient.class)
	static class ApacheHttpClientConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public ApacheHttpClientConnectionManagerFactory connManFactory() {
			return new DefaultApacheHttpClientConnectionManagerFactory();
		}

		@Bean
		@ConditionalOnMissingBean
		public HttpClientBuilder apacheHttpClientBuilder() {
			return HttpClientBuilder.create();
		}

		@Bean
		@ConditionalOnMissingBean
		public ApacheHttpClientFactory apacheHttpClientFactory(
				HttpClientBuilder builder) {
			return new DefaultApacheHttpClientFactory(builder);
		}

	}
....
....
....    
```



### 控制日志等级

​	配置`loggerlevel bean`。`FeignFactory`创建时，会查找`Logger.Level`的实例。

```java
@Bean
public feign.Logger.Level level() {
    return feign.Logger.Level.FULL;
}

```





## FeignClient接口方法上的注解解析过程

​	代码里我们看到`spring-cloud-openfeign`没有使用feign提供的`Contract`，而是自己写了一个`SpringMvcContract`，所以对接口的解析已经不是原来的思路了，使用的注解也不一样。下面做简单介绍。



首先是解析类上的注解，这里会解析类上`RequestMapping`的value值作为url属性。

```java
	@Override
	protected void processAnnotationOnClass(MethodMetadata data, Class<?> clz) {
		if (clz.getInterfaces().length == 0) {
			RequestMapping classAnnotation = findMergedAnnotation(clz,
					RequestMapping.class);
			if (classAnnotation != null) {
				// Prepend path from class annotation if specified
				if (classAnnotation.value().length > 0) {
					String pathValue = emptyToNull(classAnnotation.value()[0]);
					pathValue = resolve(pathValue);
					if (!pathValue.startsWith("/")) {
						pathValue = "/" + pathValue;
					}
					data.template().uri(pathValue);
				}
			}
		}
	}
```



然后是解析方法上的注解

```java
@Override
protected void processAnnotationOnMethod(MethodMetadata data,
                                         Annotation methodAnnotation, Method method) {
    //1.方法上只能出现RequestMapping。或者getMapper postmapper这种
    if (!RequestMapping.class.isInstance(methodAnnotation) && !methodAnnotation
        .annotationType().isAnnotationPresent(RequestMapping.class)) {
        return;
    }

    RequestMapping methodMapping = findMergedAnnotation(method, RequestMapping.class);
    // 解析请求method
    RequestMethod[] methods = methodMapping.method();
    if (methods.length == 0) {
        methods = new RequestMethod[] { RequestMethod.GET };
    }
    checkOne(method, methods, "method");
    data.template().method(Request.HttpMethod.valueOf(methods[0].name()));

    // 解析path，这里将url append进去，拼接在类上解析的url后面
    if (methodMapping.value().length > 0) {
        String pathValue = emptyToNull(methodMapping.value()[0]);
        if (pathValue != null) {
            pathValue = resolve(pathValue);
            // Append path from @RequestMapping if value is present on method
            if (!pathValue.startsWith("/") && !data.template().path().endsWith("/")) {
                pathValue = "/" + pathValue;
            }
            data.template().uri(pathValue, true);
        }
    }

    // produces
    parseProduces(data, method, methodMapping);

    // consumes
    parseConsumes(data, method, methodMapping);

    // headers
    parseHeaders(data, method, methodMapping);

    data.indexToExpander(new LinkedHashMap<Integer, Param.Expander>());
}
```



最后是解析参数上的注解

```java
@Override
	protected boolean processAnnotationsOnParameter(MethodMetadata data,
			Annotation[] annotations, int paramIndex) {
		boolean isHttpAnnotation = false;

		AnnotatedParameterProcessor.AnnotatedParameterContext context = new SimpleAnnotatedParameterContext(
				data, paramIndex);
		Method method = this.processedMethods.get(data.configKey());
		for (Annotation parameterAnnotation : annotations) {
			AnnotatedParameterProcessor processor = this.annotatedArgumentProcessors
					.get(parameterAnnotation.annotationType());
			if (processor != null) {
				Annotation processParameterAnnotation;
				// synthesize, handling @AliasFor, while falling back to parameter name on
				// missing String #value():
				processParameterAnnotation = synthesizeWithMethodParameterNameAsFallbackValue(
						parameterAnnotation, method, paramIndex);
				isHttpAnnotation |= processor.processArgument(context,
						processParameterAnnotation, method);
			}
		}

		if (isHttpAnnotation && data.indexToExpander().get(paramIndex) == null) {
			TypeDescriptor typeDescriptor = createTypeDescriptor(method, paramIndex);
			if (this.conversionService.canConvert(typeDescriptor,
					STRING_TYPE_DESCRIPTOR)) {
				Param.Expander expander = this.convertingExpanderFactory
						.getExpander(typeDescriptor);
				if (expander != null) {
					data.indexToExpander().put(paramIndex, expander);
				}
			}
		}
		return isHttpAnnotation;
	}
```

​	参数上的解析不是在此类里做的，而是委托给`annotatedArgumentProcessors`，来做，默认有下面四个解析器，分别可以解析`@PathVariable`， `@RequestParam` ，`@RequestHeader，` `@SpringQueryMap`注解。这里不展开将这四个解析器的源码了。

```java
annotatedArgumentResolvers.add(new PathVariableParameterProcessor());
annotatedArgumentResolvers.add(new RequestParamParameterProcessor());
annotatedArgumentResolvers.add(new RequestHeaderParameterProcessor());
annotatedArgumentResolvers.add(new QueryMapParameterProcessor());
```



### 总结

​		上面就是`SpringMvcContract`解析接口的过程，`spring-cloud-openfeign`抛弃了原来feign的注解，而是复用了`springmvc`里面的部分注解，降低了理解成本还是挺好的。



## 总结

​		

​		以上就是`spring-cloud`配置`openfeign`的过程，读懂源码使用起来才能得心应手。从源码看出我们只要提供好底层依赖，依靠`spring-cloud`的默认配置就能的到一个不错效果。

​		其实`spring-cloud`只使用了`openFeign`少部分自带功能，很多都是自己定制的如`encoder decoder contract`。还有一些组件是直接选择了固定的值如**retry选择了no_retry，log选择了slf4j-feign**等，主要是使用了Feign的思想而不是他的代码。



​		下篇文章我们讲一讲`spring-cloud-Feign`负载均衡相关的源码，和`spring-cloud-Feign组合断路器`相关的源码。





