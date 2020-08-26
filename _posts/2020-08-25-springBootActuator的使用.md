---
layout:     post
title:      "springBootActuator的使用"
subtitle:   ""
date:       2020-08-25
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springbootActuator
- springbootadmin
- springboot
typora-root-url: ..
---

* TOC
{:toc}


# springBootActuator的使用

## 启用actuator功能


​			这个是`Springboot`提供的分析软件，他能提供一些指数供我们参考，因为他已经内置在`Springboot`里面了，要使用它非常简单，我们只需添加如下依赖在项目下。

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
            <version>2.3.0</version>
        </dependency>
```



​		他能提供`jmx`或者`web`两种方式的数据暴露方式，我们启用基于`web`的数据暴露方式。

添加依赖后在配置文件里配置如下

```properties
#默认开启
management.endpoints.enabled-by-default=true
#以web方式暴露所有端点（include * 表示包含全部）
management.endpoints.web.exposure.include=*
#关闭jmx的所有端点（exclude * 表示排除全部）
management.endpoints.jmx.exposure.exclude=*
#health接口包含细节信息
management.endpoint.health.show-details=always
```



​		这样运行项目后，我们访问`http://127.0.0.1:8080/actuator`就能看到一个`json`，里面包含所有暴露出来的端点地址信息，但是只看`json`是不利于观察的，下面我们讲解如何配置可视化界面。





## 配置可视化界面

​		上面我们已经启用了`actuator`功能，但是数据不利于观察，于是有人做出了这样的`gui`项目，只需要将地址配置给`gui`项目，`gui`项目帮我们请求地址，将`json`数据以可视化的方式展示出来。

​	

​		在上面的项目上添加如下依赖，这是`spring-boot-admin`项目的客户端。

```xml
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-client</artifactId>
        <version>2.3.0</version>
    </dependency>
```

​	 	配置文件里配置服务端的地址，假设服务端启动在8888端口

```properties
spring.boot.admin.client.instance.service-url=http://127.0.0.1:8888
```

​			



**新建一个项目作为服务端**，添加如下依赖，这是`spring-boot-admin`项目的服务端。启动类上添加`@EnableAdminServer`注解，在8888端口启动。

```xml
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
    </dependency>
```

​		假设服务端启动在8888端口的，在客户端里面配置服务端的地址，这样客户端启动时，会连接服务端，将当前项目的`actuator`地址告诉服务端，打开`http://127.0.0.1:8888`就能展示可视化的客户端信息了。	





​		原理是这样的，拥有客户端的项目在启动时，将客户端的地址发送到服务端。服务端提供静态页面，页面上通过js访问服务端，服务端再请求项目，服务端在页面和客户端之间做转发操作。





## 实现原理源码分析

​		配置了简单的`demo`，并简述了他的原理，下面我们深入到代码里面查看他的源码。



### 客户端原理

​		找到客户端`SpringBootAdminClientAutoConfiguration`类，此类在`spring.factorise`里配置了，启动时会自动执行。我么分析他的源码即可。



```java
    @Bean
    @ConditionalOnMissingBean
    public RegistrationApplicationListener registrationListener(ClientProperties client,
                                                                ApplicationRegistrator registrator) {
        RegistrationApplicationListener listener = new RegistrationApplicationListener(registrator);
        listener.setAutoRegister(client.isAutoRegistration());
        listener.setAutoDeregister(client.isAutoDeregistration());
        listener.setRegisterPeriod(client.getPeriod());
        return listener;
    }
```



​		查看此类，其中有方法上标记了`@EventListener`注解，他在监听到容器启动完成后向服务端注册自己，下面代码里开启的注册的定时任务，10秒钟心跳注册一次。

```java
    @EventListener
    @Order(Ordered.LOWEST_PRECEDENCE)
    public void onApplicationReady(ApplicationReadyEvent event) {
        if (autoRegister) {
            startRegisterTask();
        }
    }
```

​	

​		注册逻辑是写在此方法里的，其中有一个`register`方法，负责向服务端发送信息。

```java
    @Bean
    @ConditionalOnMissingBean
    public ApplicationRegistrator registrator(ClientProperties client, ApplicationFactory applicationFactory) {
        RestTemplateBuilder builder = new RestTemplateBuilder().setConnectTimeout(client.getConnectTimeout())
                                                               .setReadTimeout(client.getReadTimeout());
        if (client.getUsername() != null) {
            builder = builder.basicAuthentication(client.getUsername(), client.getPassword());
        }
        return new ApplicationRegistrator(builder.build(), client, applicationFactory);
    }
```

​		

​		这是在`servlet`环境下的处理逻辑，可看淡它使用`RestTemplate`向服务端发送客户端的信息，如果是`webflux`环境下，会使用`webclient`发送信息。

```java
protected boolean register(Application self, String adminUrl, boolean firstAttempt) {
        try {
            ResponseEntity<Map<String, Object>> response = template.exchange(adminUrl, HttpMethod.POST,
                new HttpEntity<>(self, HTTP_HEADERS), RESPONSE_TYPE);

            if (response.getStatusCode().is2xxSuccessful()) {
                if (registeredId.compareAndSet(null, response.getBody().get("id").toString())) {
                    LOGGER.info("Application registered itself as {}", response.getBody().get("id").toString());
                } else {
                    LOGGER.debug("Application refreshed itself as {}", response.getBody().get("id").toString());
                }
                return true;
            } else {
                if (firstAttempt) {
                    LOGGER.warn(
                        "Application failed to registered itself as {}. Response: {}. Further attempts are logged on DEBUG level",
                        self, response.toString());
                } else {
                    LOGGER.debug("Application failed to registered itself as {}. Response: {}", self,
                        response.toString());
                }
            }
        } catch (Exception ex) {
            if (firstAttempt) {
                LOGGER.warn(
                    "Failed to register application as {} at spring-boot-admin ({}): {}. Further attempts are logged on DEBUG level",
                    self, client.getAdminUrl(), ex.getMessage());
            } else {
                LOGGER.debug("Failed to register application as {} at spring-boot-admin ({}): {}", self,
                    client.getAdminUrl(), ex.getMessage());
            }

        }
        return false;
    }

```







​		那么到底向服务端发送的是什么内容呢？，查看restTemplate发送请求的代码，他发送了`Application`序列化的结果。下面是一个`ApplicationFactory`，他的`createApplication`会创建一个`Application`的实例，该实例将包含服务端需要的信息。 

```java
    public static class ServletConfiguration {
        @Bean
        @ConditionalOnMissingBean
        public ApplicationFactory applicationFactory(InstanceProperties instance,
                                                     ManagementServerProperties management,
                                                     ServerProperties server,
                                                     ServletContext servletContext,
                                                     PathMappedEndpoints pathMappedEndpoints,
                                                     WebEndpointProperties webEndpoint,
                                                     MetadataContributor metadataContributor,
                                                     DispatcherServletPath dispatcherServletPath) {
            return new ServletApplicationFactory(instance,
                management,
                server,
                servletContext,
                pathMappedEndpoints,
                webEndpoint,
                metadataContributor,
                dispatcherServletPath
            );
        }
    }
```



​	

​	这就是Application内含有的字段。这些字段如果不自己配置，都可以从程序里自动获取到。

```java
public class Application {
    private final String name;
    private final String managementUrl;
    private final String healthUrl;
    private final String serviceUrl;
    private final Map<String, String> metadata;
```





#### 总结

​			这就是客户端逻辑，他先是使用监听器监听程序启动，再收集`Application`类字段上的那些信息，根据容器的不同选择`resttemplate` 或者 `webclient`，使用`Schedult`线程池，每10秒向服务器发送本程序的信息。

​			我们只需要配置服务端的地址，剩下的数据都能自动收集到。并且客户端的代码很少，就十几个类。功能也比较简单。







### 服务端原理

​		引入下面依赖后，共有三个依赖项目被引入。

```xml
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-server</artifactId>
        </dependency>
```



分别是

- spring-boot-admin-server-ui

- spring-boot-admin-server-cloud

- spring-boot-admin-server



#### spring-boot-admin-server-ui

​		首先查看`spring-boot-admin-server-ui`项目，他在项目的`META-INF`文件夹下放置了页面的静态文件，并在代码里配置了`UiController`来返回这些静态页面。页面路径相关的配置由`AdminServerUiProperties`类配置，比较简单。



#### spring-boot-admin-server-cloud

​		发现他配置了`InstanceDiscoveryListener`类，这里面对服务发现的事件进行监听，比如监听到`InstanceRegisteredEvent`事件时，调用`discoveryClient`读取注册中心，这些事件和注册中心都是在`spring-cloud-commons`包里定义的。如果使用注册中心，`admin-server`就能从注册中心拉取客户端信息，即使客户端不添加`admin-starter-client`也是可以的，这个稍后讲。

​		下面这个方法就是就是发现逻辑，使用`discoveryClient`发现服务，进行转换后存储在`InstanceRegistry`里面。

```java
    protected void discover() {
        log.debug("Discovering new instances from DiscoveryClient");
        Flux.fromIterable(discoveryClient.getServices())
            .filter(this::shouldRegisterService)
            .flatMapIterable(discoveryClient::getInstances)
            .flatMap(this::registerInstance)
            .collect(Collectors.toSet())
            .flatMap(this::removeStaleInstances)
            .subscribe(v -> { }, ex -> log.error("Unexpected error.", ex));
    }
```



​			

#### spring-boot-admin-server

首先spring.factories文件里面引入了 `AdminServerAutoConfiguration`配置类

```java
@ImportAutoConfiguration({ AdminServerInstanceWebClientConfiguration.class, AdminServerWebConfiguration.class })
public class AdminServerAutoConfiguration {
    .
    .
    .
}        
    
```



同时这个类又引入了`AdminServerInstanceWebClientConfiguration.class`，`AdminServerWebConfiguration.class`两个类。





##### AdminServerInstanceWebClientConfiguration配置类

这个类里面主要定义和配置了`InstanceWebClient.Builder`，这是一个包装了WebClient的客户端，使用它来获取客户端信息。

```java
@Configuration(proxyBeanMethods = false)
@Lazy(false)
public class AdminServerInstanceWebClientConfiguration {

	private final InstanceWebClient.Builder instanceWebClientBuilder;

//构造方法里，传入了WebClient和InstanceWebClientCustomizer，webclient.build就是springboot自动配置的 
	public AdminServerInstanceWebClientConfiguration(ObjectProvider<InstanceWebClientCustomizer> customizers,
			WebClient.Builder webClient) {
		this.instanceWebClientBuilder = InstanceWebClient.builder(webClient);
		customizers.orderedStream().forEach((customizer) -> customizer.customize(this.instanceWebClientBuilder));
	}

//这里在返回InstanceWebClient时进行了clone操作，同时也会可用一份WebClient，所以对此webclient的改动不会反应到全局webclient上    
	@Bean
	@ConditionalOnMissingBean
	@Scope("prototype")
	public InstanceWebClient.Builder instanceWebClientBuilder() {
		return this.instanceWebClientBuilder.clone();
	}
```



​		此配置类执行完成后，我们就拥有了一个封装好的`Webclient`供我们使用，并且对此类的所有配置都不会影响全局`Webclient`配置。

​		下面这个类可以为请求添加Basic验证的请求头，如果客户端需要Basic验证，并且配置了密码，就能验证访问。

```java
	@Configuration(proxyBeanMethods = false)
	protected static class HttpHeadersProviderConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public BasicAuthHttpHeaderProvider basicAuthHttpHeadersProvider() {
			return new BasicAuthHttpHeaderProvider();
		}

	}
```





##### AdminServerWebConfiguration

此类配置了一些`Controller`。如下面这个方法用来接收客户端的注册。

```java
	@PostMapping(path = "/instances", consumes = MediaType.APPLICATION_JSON_VALUE)
	public Mono<ResponseEntity<Map<String, InstanceId>>> register(@RequestBody Registration registration,
			UriComponentsBuilder builder) {
		Registration withSource = Registration.copyOf(registration).source("http-api").build();
		LOGGER.debug("Register instance {}", withSource);
		return registry.register(withSource).map((id) -> {
			URI location = builder.replacePath("/instances/{id}").buildAndExpand(id).toUri();
			return ResponseEntity.created(location).body(Collections.singletonMap("id", id));
		});
	}

```



#####  AdminServerAutoConfiguration

​	上面通过`controller`可以接受用户注册，通过`Discover`能从注册中心发现服务，那么将这些信息存储在哪里呢？

由此类里定义的三个组件完成的，分别是

- `InstanceRegistry` 客户端实例的注册器，处理注册操作
- `InstanceIdGenerator` 客户端id生成
- `SnapshottingInstanceRepository` 上面的注册器实际上将信息存储在这个类里面，相当于是注册器的数据库，默认存储在内存里。





#### 服务发现和密码保护

​			客户端`Actuator`相关接口一般是要加密的，如果采用HttpBasic方式加密，可以将密码配置到服务端，或者配置到注册中心的元数据里面，默认`admin-server`会从元数据里面获取如下的帐号密码。

```java

	private static final String[] USERNAME_KEYS = { "user.name", "user-name", "username" };

	private static final String[] PASSWORD_KEYS = { "user.password", "user-password", "userpassword" };

```



​			eureka注册中心将数据存储到元数据里面的方法是下面这样，这样服务端就不用配置客户端的密码了，能直接从注册中心里获取到。

```properties
eureka.instance.metadata-map.user-name=abc
eureka.instance.metadata-map.user-password=abc
```



#### 总结

​		以上就是`admin-server`端的源码分析，他能使用服务发现，或者接口的方式得到客户端的信息，并存储起来，如果管理网页被打开，就会进行请求转发，转发到对应客户端的地址上。如果不存在注册中心，他就自己维持心跳，维持状态，否则它可以直接从注册中心内获取这些信息。关于密码保护，如果使用注册中心的话，可以将密码存储到元数据里，他尝试从元数据里按照上面三个名字获取账号密码。



### 总结

​		以上就是`Actuator`可视化的过程，强烈推荐将`admin-server`单独部署为一个项目，将所有项目部署到一个注册中心里，这样其他项目不使用`admin-client`也是可以的
