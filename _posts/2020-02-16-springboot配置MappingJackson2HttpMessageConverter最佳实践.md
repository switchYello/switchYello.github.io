---

layout:     post
title:      "springboot配置MappingJackson2HttpMessageConverter最佳实践"
subtitle:   ""
date:       2020-02-16
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- Jackson
- HttpMessageConverter
- Json
typora-root-url: ..
---



​	

上篇讲了，`SpringMvc是如何将返回值转成Json的`，

默认情况下`Springboot是如何配置HandlerAdapter`的，如何根据配置文件影响到`ObjectMapper`的创建的。

下面讲一讲如何做才是最佳实践，如何做能满足需求。



## 1. 最大限度使用`Springboot`的自动配置

如果我们想最大限度使用`Springboot`，且想修改接口返回`Json`格式等，那么我们可以在配置文件中配置常用的配置。

我们在配置文件里打出`spring.jackson`，`ide`会给我们提示，我们发现常用的配置都有，比如配置输出时间格式的

```
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
```



也可以打开看看下面这个类,里面也有对应的配置

```java
org.springframework.boot.autoconfigure.jackson.JacksonProperties
```



## 2.使用自己创建的`ObjectMapper`

创建一个`ObjectMapper`的`Bean`，但要标记为`@Primary`才能覆盖自动配置的。这样的话配置文件里的配置不会生效，我们自己创建的就可以随心所欲的配置了。

并且推荐使用下面的第二种方式创建，点开源码能看到它里面添加了额外的配置。

```java
    @Bean
    @Primary
    public ObjectMapper objectMapper (){
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd"));
        return objectMapper;
    }
```



```java
    @Bean
    @Primary
    public ObjectMapper objectMapper (){
        return Jackson2ObjectMapperBuilder
                .json()
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
                .build();
    }
```

因为`org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration`里的自动配置，加入了`@ConditionalOnMissingBean`注解，所以系统就不会自动配置了。





## 3. 使用自己创建的`MappingJackson2HttpMessageConverter`

```java
    @Bean
    public MappingJackson2HttpMessageConverter objectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd"));
        return new MappingJackson2HttpMessageConverter(objectMapper);
    }
```



因为`org.springframework.boot.autoconfigure.http.JacksonHttpMessageConvertersConfiguration`类中对`MappingJackson2HttpMessageConverter`的自动配置添加了条件判断，所以自动配置就不会执行了。

判断如下

```java
@ConditionalOnMissingBean(value = MappingJackson2HttpMessageConverter.class,
```



## 4.如果继承`WebMvcConfigurationSupport`

这个类里面又很多`MVC`相关的配置，我个人比较习惯继承这个类，但是继承这个类后，上面三种方法无论是改配置文件，自定义`ObjectMapper`，还是自己定义`MappingJackson2HttpMessageConverter`均失效了。

大家可以试一下，那么是什么原因导致失效的呢？，如何解决这个问题？

####  1.是什么原因导致失效的呢？

我们可以看一下`WebMvcAutoConfiguration`的源码，发现这个类上有`@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`的注解，

也就是说找不到`WebMvcConfigurationSupport`才进行自动配置，而我们继承了`WebMvcConfigurationSupport`，并添加@Configuration，则`MapperAdaper`不会被自动配置，所以自动创建的

`MappingJackson2HttpMessageConverter`和`ObjectMapper`都虽然创建了但也不会起作用了。



#### 2. 如何解决这个问题？

既然我习惯继承`WebMvcConfigurationSupport`，如果想定制`ObjectMapper`该怎么办呢，查看`WebMvcConfigurationSupport`源码,发现他有这样一个方法。

```java
	@Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
		RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
		adapter.setContentNegotiationManager(mvcContentNegotiationManager());
		//这里注入了MessageConverter，请进入getMessageConverters()方法
		adapter.setMessageConverters(getMessageConverters());
		adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer());
		adapter.setCustomArgumentResolvers(getArgumentResolvers());
		adapter.setCustomReturnValueHandlers(getReturnValueHandlers());

		if (jackson2Present) {
			adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
			adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
		}

		AsyncSupportConfigurer configurer = new AsyncSupportConfigurer();
		configureAsyncSupport(configurer);
		if (configurer.getTaskExecutor() != null) {
			adapter.setTaskExecutor(configurer.getTaskExecutor());
		}
		if (configurer.getTimeout() != null) {
			adapter.setAsyncRequestTimeout(configurer.getTimeout());
		}
		adapter.setCallableInterceptors(configurer.getCallableInterceptors());
		adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

		return adapter;
	}
```



从上面源码能看到，没有在`WebMvcAutoConfiguration`里面配置的`RequestMappingHandlerAdapter`在这里配置了，但这里并没有使用Spring容器里面的`ObjectMapper`和`MappingJackson2HttpMessageConverter`，而是新创建的，所以系统自动配置的和我们自己注入到`Spring容器`的才会失效，因为根本就没有被使用。



我们看一下是如何添加MessageConverter的。

```java
	protected final List<HttpMessageConverter<?>> getMessageConverters() {
		if (this.messageConverters == null) {
			this.messageConverters = new ArrayList<>();
			//这个方法是空实现
			configureMessageConverters(this.messageConverters);
			if (this.messageConverters.isEmpty()) {
			//添加默认的messageConverters
				addDefaultHttpMessageConverters(this.messageConverters);
			}
			//继续处理
			extendMessageConverters(this.messageConverters);
		}
		return this.messageConverters;
	}
```



上面代码能看出，首先是通过`configureMessageConverters(this.messageConverters)`方法配置，但该方法是空实现，然后添加默认的`MessageConverter`,然后`extendMessageConverters(this.messageConverters)`





所以我们可以在继承类中重写`extendMessageConverters`方法，在这个类中对默认的`MessageConverter`进行修改或添加删除等操作，如下。

```java

    @Override
    protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        //将StringHttpMessageConverter设成UTF8格式
        converters.stream()
                .filter(c -> c instanceof StringHttpMessageConverter)
                .map(c -> (StringHttpMessageConverter) c)
                .forEach(c -> c.setDefaultCharset(StandardCharsets.UTF_8));

        //将MappingJackson2HttpMessageConverter设为自己的ObjectMapper
        ObjectMapper mapper = new ObjectMapper();
        mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd"));
        converters.stream()
                .filter(c -> c instanceof MappingJackson2HttpMessageConverter)
                .map(c -> (MappingJackson2HttpMessageConverter) c)
                .forEach(c -> c.setObjectMapper(mapper));
    }
```



## 5.总结

如果你不打算继承`WebMvcConfigurationSupport`，那么自动配置生效，你可以通过配置文件配置`Jackson`或通过提供自己的类配置`Jackson`，但注意如果覆盖`ObjectMappe`r要加`@Primary`注解才行。



如果你不小心或打算继承`WebMvcConfigurationSupport`，那上面的自动配置依然会执行，但不会被使用，这就导致上面的配置不灵了，可以实现`extendMessageConverters`方法，在里面我们可以任意处理`MessageConverter`。



## 6.回答下上篇文章的问题

1. 配置文件里配置列`Jackson`但不生效？

可能是覆盖了`WebMvcConfigurationSupport`，或重写了`ObjectMapper`或重写了`MappingJackson2HttpMessageConverter`，这样导致配置文件里的不生效了。



2. 自己手动创建`ObjectMapper`来代替`Springboot`帮我们创建的，但不生效？

你肯定是继承了`WebMvcConfigurationSupport`。



3. 如何做才是最佳实践？

上面已经列举了四种方式，要注意他们的优先级，且继承`WebMvcConfigurationSupport`会导致前三种方式创建的对象不被使用，也就是失效了。