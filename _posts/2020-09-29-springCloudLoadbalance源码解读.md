---
layout:     post
title:      "springCloudLoadbalance源码解读"
subtitle:   ""
date:       2020-09-29
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring
- springcloud
- 负载均衡
- loadbalance
typora-root-url: ..
---



# springCloudLoadbalance源码解读



​		本文讲解`spring-cloud-loadbalance`这个包，这个包提供了微服务下的负载均衡功能。要想使用负载均衡前提是必须拥有注册中心，同一个名字的服务在注册中心上存在多个实例，这是就需要负载均衡来实现对多个实例的均衡请求。

​		类比一下dns系统，一个域名在dns上可能存在多个ip地址，这就相当于同一个微服务模块在注册中心上存在多个实例，我们进行dns查询后获取到的多个ip地址，我们会选择其中一个进行连接，具体如何选择就需要一种策略来实现。一般查询dns会选择第一个ip地址，而微服务里的负载均衡会采用轮询的方式平均使用所有的实例。



### 堵塞式负载均衡接口api

​		

​		首先看它的继承图

![image-20200930163757870](/img/in/2020-09-29-springCloudLoadbalance/image-20200930163757870.png)



​		负载均衡的接口是这样的，他的核心接口是`LoadBalancerClient`。

```java
public interface ServiceInstanceChooser {
    //传入微服务id，返回服务的详细信息（ip地址，端口 等）
	ServiceInstance choose(String serviceId);
    
}
```



```java
public interface LoadBalancerClient extends ServiceInstanceChooser {
	
    //传入微服务id和request对象，将request对象的执行返回结果返回
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

    //传入微服务id，微服务实例，request，将request对象的执行结果返回
	<T> T execute(String serviceId, ServiceInstance serviceInstance,
			LoadBalancerRequest<T> request) throws IOException;

    //传入服务信息，和原始uri，将原始uri里面的host字段替换成真实服务的ip并返回
	URI reconstructURI(ServiceInstance instance, URI original);

}

```



而`LoadBalancerRequest`对象像下面这样，只需要传入微服务实例，就能得到结果。

```java
public interface LoadBalancerRequest<T> {

	T apply(ServiceInstance instance) throws Exception;

}
```



​		所以核心方法就是`choose(String serviceId)`，他能由字符串形式的服务id得到服务实例。`LoadBalancerClient`的默认实现是这样的。我已经将多余部分删除了，他的逻辑就是使用`choose`方法获取到服务实例，然后调用`request`对象的apply方法就能得到结果。

```java
public class BlockingLoadBalancerClient implements LoadBalancerClient {
	
	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
			throws IOException {
		ServiceInstance serviceInstance = choose(serviceId);
		return execute(serviceId, serviceInstance, request);
	}

	@Override
	public <T> T execute(String serviceId, ServiceInstance serviceInstance,
			LoadBalancerRequest<T> request) throws IOException {
			return request.apply(serviceInstance);
	}

	@Override
	public URI reconstructURI(ServiceInstance serviceInstance, URI original) {
		return LoadBalancerUriTools.reconstructURI(serviceInstance, original);
	}
    
    
    @Override
        public ServiceInstance choose(String serviceId) {
            ReactiveLoadBalancer<ServiceInstance> loadBalancer = loadBalancerClientFactory.getInstance(serviceId);
            if (loadBalancer == null) {
                return null;
            }
            Response<ServiceInstance> loadBalancerResponse = Mono.from(loadBalancer.choose()).block();
            if (loadBalancerResponse == null) {
                return null;
            }
            return loadBalancerResponse.getServer();
        }
    

```



​		可以看到核心方法`choose`里面的逻辑是委托给`ReactiveLoadBalancer`来实现的，这是一种常见做法，将堵塞式的方法委托给非堵塞式的方法。下面看看这个非堵塞式接口是如何实现的。





### 非堵塞式api

​	上面是堵塞式的方式，而这种纯网络请求，不落库的场景是很容易做成异步形式的，下面看看非堵塞方式的api骨架。

​		

![image-20200930163902482](/img/in/2020-09-29-springCloudLoadbalance/image-20200930163902482.png)



```java
public interface ReactorLoadBalancer<T> extends ReactiveLoadBalancer<T> {

	Mono<Response<T>> choose(Request request);

	default Mono<Response<T>> choose() {
		return choose(REQUEST);
	}

}
```



非堵塞式的api更简单，他有一个`choose`方法，返回值是`Response<T>`，这个泛型T就是想要的到的结果数据。

我们要做的事情就是由服务id，选取一个服务实例返回，这就是choose方法应该做到的。





### 具体实现

​	上面的就是`spring-cloud`负载均衡方面的接口定义，具体实现都在`spring-cloud-loadbalance`依赖里面。

打开spring.factories文件

![image-20200930141712970](/img/in/2020-09-29-springCloudLoadbalance/image-20200930141712970.png)



这个包下的配置文件很简单，只有三个，下面一个一个分析。





### 1. LoadBalancerCacheAutoConfiguration

​		很显然此类与缓存有关，我们从注册中心获取到的服务信息是要缓存起来的，不可能每次都重新获取，此类上会创建一个`LoadBalancerCacheManager`对象，底层根据当前chasspath来选择Caffeine或evictor来缓存实现。

```java
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(Caffeine.class)
	protected static class CaffeineLoadBalancerCacheManagerConfiguration {

		@Bean(autowireCandidate = false)
		@ConditionalOnMissingBean
		LoadBalancerCacheManager caffeineLoadBalancerCacheManager(
				LoadBalancerCacheProperties cacheProperties) {
			return new CaffeineBasedLoadBalancerCacheManager(cacheProperties);
		}

	}
```



### 2. BlockingLoadBalancerClientAutoConfiguration

​		此类也很简单，他创建了`BlockingLoadBalancerClient`的对象，并且传入`LoadBalancerClientFactory`，因为堵塞式的处理逻辑是委托给非堵塞式的，这里没有太多逻辑。

```java
    @Bean
    @ConditionalOnBean(LoadBalancerClientFactory.class)
    @Primary
    public BlockingLoadBalancerClient blockingLoadBalancerClient(
        LoadBalancerClientFactory loadBalancerClientFactory) {
        return new BlockingLoadBalancerClient(loadBalancerClientFactory);
    }
```

​	

​		此堵塞式的客户端会将所有逻辑委托给非堵塞式的客户端。

```java
	public ServiceInstance choose(String serviceId) {
		//获取非堵塞式客户端
        ReactiveLoadBalancer<ServiceInstance> loadBalancer = loadBalancerClientFactory.getInstance(serviceId);
		if (loadBalancer == null) {
			return null;
		}
        //将逻辑委托给非堵塞客户端
		Response<ServiceInstance> loadBalancerResponse = Mono.from(loadBalancer.choose()).block();
		if (loadBalancerResponse == null) {
			return null;
		}
		return loadBalancerResponse.getServer();
	}
```





### 3. LoadBalancerAutoConfiguration

​		此方法创建`LoadBalancerClientFactory`对象，这个类是一个`NamedContextFactory`类，他能根据当前容器创建子容器。

```java
	@ConditionalOnMissingBean
	@Bean
	public LoadBalancerClientFactory loadBalancerClientFactory() {
		LoadBalancerClientFactory clientFactory = new LoadBalancerClientFactory();
		clientFactory.setConfigurations(
				this.configurations.getIfAvailable(Collections::emptyList));
		return clientFactory;
	}
```



构造方法如下，默认将`LoadBalancerClientConfiguration`类注册到子容器内。

```java
public class LoadBalancerClientFactory
		extends NamedContextFactory<LoadBalancerClientSpecification>
		implements ReactiveLoadBalancer.Factory<ServiceInstance> {

	public static final String NAMESPACE = "loadbalancer";


	public static final String PROPERTY_NAME = NAMESPACE + ".client.name";
	
    //默认注册LoadBalancerClientConfiguration类
	public LoadBalancerClientFactory() {
		super(LoadBalancerClientConfiguration.class, NAMESPACE, PROPERTY_NAME);
	}

	public String getName(Environment environment) {
		return environment.getProperty(PROPERTY_NAME);
	}
	
    //从容器内获取ReactiveLoadBalancer
	@Override
	public ReactiveLoadBalancer<ServiceInstance> getInstance(String serviceId) {
		return getInstance(serviceId, ReactorServiceInstanceLoadBalancer.class);
	}
}

```



主要看他的`getInstance`方法

```java
	public <T> T getInstance(String name, Class<T> type) {
        //获取子容器
		AnnotationConfigApplicationContext context = getContext(name);
		//从子容器内获取type类型的bean
        if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
				type).length > 0) {
			return context.getBean(type);
		}
		return null;
	}
```



获取子容器的过程，这里使用缓存存储创建的每个子容器，子容器不会重复创建

```java
	protected AnnotationConfigApplicationContext getContext(String name) {
		if (!this.contexts.containsKey(name)) {
			synchronized (this.contexts) {
				if (!this.contexts.containsKey(name)) {
					this.contexts.put(name, createContext(name));
				}
			}
		}
		return this.contexts.get(name);
	}
```



创建子容器的过程

```java
	protected AnnotationConfigApplicationContext createContext(String name) {
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        //如果存在name名称的配置将配置注册进去
        //这里的配置可以通过setConfigurations方法设置
		if (this.configurations.containsKey(name)) {
			for (Class<?> configuration : this.configurations.get(name)
					.getConfiguration()) {
				context.register(configuration);
			}
		}
        
        //查询是否有default.开头的配置，将这些配置注册进去
		for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
			if (entry.getKey().startsWith("default.")) {
				for (Class<?> configuration : entry.getValue().getConfiguration()) {
					context.register(configuration);
				}
			}
		}
      
        //注册构造方法里传入的LoadBalancerClientConfiguration.class
		context.register(PropertyPlaceholderAutoConfiguration.class,
				this.defaultConfigType);
        
          //将[propertyName - name] 存入环境变量
		context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
				this.propertySourceName,
				Collections.<String, Object>singletonMap(this.propertyName, name)));
		if (this.parent != null) {
			//将子容器的父容器设为当前容器
			context.setParent(this.parent);
			context.setClassLoader(this.parent.getClassLoader());
		}
		context.setDisplayName(generateDisplayName(name));
		context.refresh();
		return context;
	}
```

​		

​	下面是`LoadBalancerClientConfiguration`配置类，每个`serverId`都会创建一个子容器，每个子容器都会解析一遍此类，将解析出来的bean注册进容器。

```java
public class LoadBalancerClientConfiguration {

	private static final int REACTIVE_SERVICE_INSTANCE_SUPPLIER_ORDER = 193827465;

	@Bean
	@ConditionalOnMissingBean
	public ReactorLoadBalancer<ServiceInstance> reactorServiceInstanceLoadBalancer(
			Environment environment,
			LoadBalancerClientFactory loadBalancerClientFactory) {
		//从容器内获取serverId名称
        //因为创建子容器过程中，已经将[propertyName - name]键值对存储环境变量里了，此处获取的就是serverId
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
		
        //从容器内获取ServiceInstanceListSupplier对象，这是ServiceInstance的列表
        return new RoundRobinLoadBalancer(loadBalancerClientFactory.getLazyProvider(name,
				ServiceInstanceListSupplier.class), name);
	}
    
 /*
 下面这些就是配置ServiceInstanceListSupplier的过程，他会根据当前环境自动选择堵塞式或者非堵塞式
 使用order注解控制顺序，非堵塞式的优先堵塞式的。
 
 同时根据配置文件选择是否根据地域优先或使用health-check
 */   
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnReactiveDiscoveryEnabled
	@Order(REACTIVE_SERVICE_INSTANCE_SUPPLIER_ORDER)
	public static class ReactiveSupportConfiguration {

		@Bean
		@ConditionalOnBean(ReactiveDiscoveryClient.class)
		@ConditionalOnMissingBean
		@ConditionalOnProperty(value = "spring.cloud.loadbalancer.configurations",
				havingValue = "default", matchIfMissing = true)
		public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
				ConfigurableApplicationContext context) {
			return ServiceInstanceListSupplier.builder().withDiscoveryClient()
					.withCaching().build(context);
		}

		@Bean
		@ConditionalOnBean(ReactiveDiscoveryClient.class)
		@ConditionalOnMissingBean
		@ConditionalOnProperty(value = "spring.cloud.loadbalancer.configurations",
				havingValue = "zone-preference")
		public ServiceInstanceListSupplier zonePreferenceDiscoveryClientServiceInstanceListSupplier(
				ConfigurableApplicationContext context) {
			return ServiceInstanceListSupplier.builder().withDiscoveryClient()
					.withZonePreference().withCaching().build(context);
		}

		@Bean
		@ConditionalOnBean(ReactiveDiscoveryClient.class)
		@ConditionalOnMissingBean
		@ConditionalOnProperty(value = "spring.cloud.loadbalancer.configurations",
				havingValue = "health-check")
		public ServiceInstanceListSupplier healthCheckDiscoveryClientServiceInstanceListSupplier(
				ConfigurableApplicationContext context) {
			return ServiceInstanceListSupplier.builder().withDiscoveryClient()
					.withHealthChecks().withCaching().build(context);
		}

		@Bean
		@ConditionalOnBean(ReactiveDiscoveryClient.class)
		@ConditionalOnMissingBean
		public ServiceInstanceSupplier discoveryClientServiceInstanceSupplier(
				ReactiveDiscoveryClient discoveryClient, Environment env,
				ApplicationContext context) {
			DiscoveryClientServiceInstanceSupplier delegate = new DiscoveryClientServiceInstanceSupplier(
					discoveryClient, env);
			ObjectProvider<LoadBalancerCacheManager> cacheManagerProvider = context
					.getBeanProvider(LoadBalancerCacheManager.class);
			if (cacheManagerProvider.getIfAvailable() != null) {
				return new CachingServiceInstanceSupplier(delegate,
						cacheManagerProvider.getIfAvailable());
			}
			return delegate;
		}

	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnBlockingDiscoveryEnabled
	@Order(REACTIVE_SERVICE_INSTANCE_SUPPLIER_ORDER + 1)
	public static class BlockingSupportConfiguration {

		@Bean
		@ConditionalOnBean(DiscoveryClient.class)
		@ConditionalOnMissingBean
		@ConditionalOnProperty(value = "spring.cloud.loadbalancer.configurations",
				havingValue = "default", matchIfMissing = true)
		public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
				ConfigurableApplicationContext context) {
			return ServiceInstanceListSupplier.builder().withBlockingDiscoveryClient()
					.withCaching().build(context);
		}

		@Bean
		@ConditionalOnBean(DiscoveryClient.class)
		@ConditionalOnMissingBean
		@ConditionalOnProperty(value = "spring.cloud.loadbalancer.configurations",
				havingValue = "zone-preference")
		public ServiceInstanceListSupplier zonePreferenceDiscoveryClientServiceInstanceListSupplier(
				ConfigurableApplicationContext context) {
			return ServiceInstanceListSupplier.builder().withBlockingDiscoveryClient()
					.withZonePreference().withCaching().build(context);
		}

		@Bean
		@ConditionalOnBean(DiscoveryClient.class)
		@ConditionalOnMissingBean
		@ConditionalOnProperty(value = "spring.cloud.loadbalancer.configurations",
				havingValue = "health-check")
		public ServiceInstanceListSupplier healthCheckDiscoveryClientServiceInstanceListSupplier(
				ConfigurableApplicationContext context) {
			return ServiceInstanceListSupplier.builder().withBlockingDiscoveryClient()
					.withHealthChecks().withCaching().build(context);
		}

		@Bean
		@ConditionalOnBean(DiscoveryClient.class)
		@ConditionalOnMissingBean
		public ServiceInstanceSupplier discoveryClientServiceInstanceSupplier(
				DiscoveryClient discoveryClient, Environment env,
				ApplicationContext context) {
			DiscoveryClientServiceInstanceSupplier delegate = new DiscoveryClientServiceInstanceSupplier(
					discoveryClient, env);
			ObjectProvider<LoadBalancerCacheManager> cacheManagerProvider = context
					.getBeanProvider(LoadBalancerCacheManager.class);
			if (cacheManagerProvider.getIfAvailable() != null) {
				return new CachingServiceInstanceSupplier(delegate,
						cacheManagerProvider.getIfAvailable());
			}
			return delegate;
		}

	}

}

```





​		下面看`RoundRobinLoadBalancer`是如何处理的。他从`serviceInstanceListSupplierProvider`里面获取到`List<ServiceInstance>`，并使用`getInstanceResponse`方法轮询选择一个服务对象返回。

```java
public class RoundRobinLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    	public Mono<Response<ServiceInstance>> choose(Request request) {
		// TODO: move supplier to Request?
		// Temporary conditional logic till deprecated members are removed.
		if (serviceInstanceListSupplierProvider != null) {
			ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
					.getIfAvailable(NoopServiceInstanceListSupplier::new);
            
            //获取服务列表
			return supplier.get().next().map(this::getInstanceResponse);
		}
		ServiceInstanceSupplier supplier = this.serviceInstanceSupplier
				.getIfAvailable(NoopServiceInstanceSupplier::new);
		return supplier.get().collectList().map(this::getInstanceResponse);
	}
    
    //使用轮询的方式
    private Response<ServiceInstance> getInstanceResponse(
        List<ServiceInstance> instances) {
        int pos = Math.abs(this.position.incrementAndGet());

        ServiceInstance instance = instances.get(pos % instances.size());

        return new DefaultResponse(instance);
    }
    
}

```





​		默认`ServiceInstanceListSupplier`的生成是有`ServiceInstanceListSupplier.builder`工具类实现的，最后调用的`build`方法将当前容器传入进去，所需的`discoverClient`，`cache`等都能从容器内获取。

```java
public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
    ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder().withDiscoveryClient()
        .withCaching().build(context);
}
```







### @LoadBalancerClient注解

​		在创建`LoadBalancerClientFactory`时注意到调用`setConfigurations`方法，这些配置类是自动注入进来的，那么这些配置类是从哪里创建的，又是在哪里起作用的呢，通过`@LoadBalancerClient`注解可以方便的注入这些配置进去。

```java
	@ConditionalOnMissingBean
	@Bean
	public LoadBalancerClientFactory loadBalancerClientFactory() {
		LoadBalancerClientFactory clientFactory = new LoadBalancerClientFactory();
		clientFactory.setConfigurations(
				this.configurations.getIfAvailable(Collections::emptyList));
		return clientFactory;
	}
```



注解需要提供名称表明配置要注册到哪个子容器里。

```java
public @interface LoadBalancerClient {

	@AliasFor("name")
	String value() default "";

	@AliasFor("value")
	String name() default "";

	Class<?>[] configuration() default {};

}

```



此注解引入了`LoadBalancerClientConfigurationRegistrar`类，这里将注解上name和configuration包装成`LoadBalancerClientSpecification`对象，注册到容器里。

```java
	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		Map<String, Object> attrs = metadata
				.getAnnotationAttributes(LoadBalancerClients.class.getName(), true);
		if (attrs != null && attrs.containsKey("value")) {
			AnnotationAttributes[] clients = (AnnotationAttributes[]) attrs.get("value");
			for (AnnotationAttributes client : clients) {
				registerClientConfiguration(registry, getClientName(client),
						client.get("configuration"));
			}
		}
		if (attrs != null && attrs.containsKey("defaultConfiguration")) {
			String name;
			if (metadata.hasEnclosingClass()) {
				name = "default." + metadata.getEnclosingClassName();
			}
			else {
				name = "default." + metadata.getClassName();
			}
			registerClientConfiguration(registry, name,
					attrs.get("defaultConfiguration"));
		}
		Map<String, Object> client = metadata
				.getAnnotationAttributes(LoadBalancerClient.class.getName(), true);
		String name = getClientName(client);
		if (name != null) {
			registerClientConfiguration(registry, name, client.get("configuration"));
		}
	}


	private static void registerClientConfiguration(BeanDefinitionRegistry registry,
			Object name, Object configuration) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder
				.genericBeanDefinition(LoadBalancerClientSpecification.class);
		builder.addConstructorArgValue(name);
		builder.addConstructorArgValue(configuration);
		registry.registerBeanDefinition(name + ".LoadBalancerClientSpecification",
				builder.getBeanDefinition());
	}
```

​	



​		创建子容器时，根据名字从列表里获取配置并注册。

```java
protected AnnotationConfigApplicationContext createContext(String name) {
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    //根据名字选择筛选出配置，将配置注册到子容器里
		if (this.configurations.containsKey(name)) {
			for (Class<?> configuration : this.configurations.get(name)
					.getConfiguration()) {
				context.register(configuration);
			}
		}
		for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
			if (entry.getKey().startsWith("default.")) {
				for (Class<?> configuration : entry.getValue().getConfiguration()) {
					context.register(configuration);
				}
			}
		}
```



上面可以看到`@LoadBalancerClient`是引入一个名字和配置类，配置类最终会注册到名称指定的子容器里。



总结一下负载均衡获取根据实例id获取实例的过程

1. 创建`LoadBalancerClientFactory`工厂类，其是一个子容器的工厂
2. `LoadBalancerClientFactory`获取bean时会自动创建子容器，创建过程中会解析配置类（构造方法里传入的和`@LoadBalancerClient`注解引入的）
3. 由工厂类获取`ReactiveLoadBalancer`对象（解析好配置类后，此对象就会在子容器里）
4. 创建`BlockingLoadBalancerClient`供堵塞式api使用，其将逻辑委托给`ReactiveLoadBalancer`
5. 这样我们就能从容器里获取堵塞式和非堵塞式的`LoadbalancerClient`了





### 总结

​		`spring-loadbalance`是一个小巧的库，它支持堵塞式和非堵塞式的api，其中堵塞式是委托给非堵塞式来完成的，它的原理是使用`DiscoverClient`获取指定服务id对应的服务实例列表，通过轮询的方式从其中选择一个实例，并且可以配置缓存。 







