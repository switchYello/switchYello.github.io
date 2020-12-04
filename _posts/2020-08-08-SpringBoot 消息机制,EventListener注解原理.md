---
layout:     post
title:      "SpringBoot @EventListener注解的原理和使用"
subtitle:   ""
date:       2020-08-08
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- "springboot"
- "观察者模式"
- "监听器模式"
- "@EventListerer注解"
- "springboot消息机制" 
---



# SpringBoot @EventListener注解的原理和使用

## 1.观察者模式/监听者模式

​		这个模式大家应该很熟悉，也经常被使用。但是如果我们使用的是`Spring`框架，其实它内置了一个好用的观察者模式的实现，用法也很简单。

## 2.用法示例

​	下面的代码，首先在类里自动注入了`ApplicationEventPublisher`，这个也就是我们的`ApplicationCOntext`，它实现了这个接口。

再在post方法里，发送一条消息，这个消息其实就是一个类的实力。这样这条消息就会发送给监听器。

```java
	// 1.自动注入ApplicationEventPublisher
	@Autowired
	private ApplicationEventPublisher applicationEventPublisher;
	
	// 2.在类里面发送消息
	@PostConstruct
	public void post(){
		applicationEventPublisher.publishEvent(new UserCreateMessage("张三"));
	}

	public static class UserCreateMessage{
		public UserCreateMessage(String name) {
			this.name = name;
		}

		private String name;
		private Date createTime;
	}
```



​		在另一个类配置类里面使用注解`@EventListener`标记一个方法，这里注意参数是泛型的`PayloadApplicationEvent`，泛型就是我们的消息类型。因为上面发送的消息会被包装成`PayloadApplicationEvent`对象发送给监听者，**只有泛型类型一致的才会匹配成功，且方法只能有一个参数**。

```java
	//3. 这个方法就可以收到消息
	@EventListener
	public void listener1(PayloadApplicationEvent<UserCreateMessage> event){
		UserCreateMessage payload = event.getPayload();
		System.out.println(payload);
	}
	
```



​	这就是最简单的用法，要保证上面的注解类会被注册到Spring里才行哦。

还有就是调用监听者是同步的，在监听方法里抛异常或堵塞都会影响发送消息的方法。



## 3.原理

其实上面添加`@EventListener`注解的方法被包装成了`ApplicationListener`对象，上面的类似于下面这种写法，这个应该比较好理解。

```java
@Component
public class MyListener implements ApplicationListener<PayloadApplicationEvent<WebApplication.UserCreateMessage>> {
	
	@Override
	public void onApplicationEvent(PayloadApplicationEvent<WebApplication.UserCreateMessage> event) {
		WebApplication.UserCreateMessage payload = event.getPayload();
		System.out.println(payload);
	}
	
}
```



​		那么Spring是什么时候做这件事的呢？

查看`SpringBoot`的源码，找到下面的代码,因为我是Tomcat环境，这里创建的`ApplicationContext`是`org.springframework.bootweb.servlet.context.AnnotationConfigServletWebServerApplicationContext`

```java
	protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				switch (this.webApplicationType) {
				case SERVLET:
					contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
					break;
				case REACTIVE:
					contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
					break;
				default:
					contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
				}
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, " + "please specify an ApplicationContextClass",
						ex);
			}
		}
		return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
```



这是他的构造方法

```java
	public AnnotationConfigServletWebServerApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
```



进到`AnnotatedBeanDefinitionReader`里面

```java
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
```





再进到`AnnotationConfigUtis`的方法里面，省略了一部分代码，可以看到他注册了一个`EventListenerMethodProcessor`类到工厂了。这是一个`BeanFactory`的后置处理器。



```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
	......
    .....
    ......    

	if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}
    
    ......
    ......

		return beanDefs;
	}
```



查看这个`BeanFactory`的后置处理器`EventListenerMethodProcessor`，下面方法，他会遍历所有bean，找到其中带有`@EventListener`的方法，将它包装成`ApplicationListenerMethodAdapter`，注册到工厂里，这样就成功注册到Spring的监听系统里了。

```java
	@Override
	public void afterSingletonsInstantiated() {
		ConfigurableListableBeanFactory beanFactory = this.beanFactory;
		Assert.state(this.beanFactory != null, "No ConfigurableListableBeanFactory set");
		String[] beanNames = beanFactory.getBeanNamesForType(Object.class);
		for (String beanName : beanNames) {
			if (!ScopedProxyUtils.isScopedTarget(beanName)) {
				Class<?> type = null;
				try {
					type = AutoProxyUtils.determineTargetClass(beanFactory, beanName);
				}
				catch (Throwable ex) {
					// An unresolvable bean type, probably from a lazy bean - let's ignore it.
					if (logger.isDebugEnabled()) {
						logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
					}
				}
				if (type != null) {
					if (ScopedObject.class.isAssignableFrom(type)) {
						try {
							Class<?> targetClass = AutoProxyUtils.determineTargetClass(
									beanFactory, ScopedProxyUtils.getTargetBeanName(beanName));
							if (targetClass != null) {
								type = targetClass;
							}
						}
						catch (Throwable ex) {
							// An invalid scoped proxy arrangement - let's ignore it.
							if (logger.isDebugEnabled()) {
								logger.debug("Could not resolve target bean for scoped proxy '" + beanName + "'", ex);
							}
						}
					}
					try {
						processBean(beanName, type);
					}
					catch (Throwable ex) {
						throw new BeanInitializationException("Failed to process @EventListener " +
								"annotation on bean with name '" + beanName + "'", ex);
					}
				}
			}
		}
	}




private void processBean(final String beanName, final Class<?> targetType) {
		if (!this.nonAnnotatedClasses.contains(targetType) &&
				!targetType.getName().startsWith("java") &&
				!isSpringContainerClass(targetType)) {

			Map<Method, EventListener> annotatedMethods = null;
			try {
				annotatedMethods = MethodIntrospector.selectMethods(targetType,
						(MethodIntrospector.MetadataLookup<EventListener>) method ->
								AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));
			}
			catch (Throwable ex) {
				// An unresolvable type in a method signature, probably from a lazy bean - let's ignore it.
				if (logger.isDebugEnabled()) {
					logger.debug("Could not resolve methods for bean with name '" + beanName + "'", ex);
				}
			}

			if (CollectionUtils.isEmpty(annotatedMethods)) {
				this.nonAnnotatedClasses.add(targetType);
				if (logger.isTraceEnabled()) {
					logger.trace("No @EventListener annotations found on bean class: " + targetType.getName());
				}
			}
			else {
				// Non-empty set of methods
				ConfigurableApplicationContext context = this.applicationContext;
				Assert.state(context != null, "No ApplicationContext set");
				List<EventListenerFactory> factories = this.eventListenerFactories;
				Assert.state(factories != null, "EventListenerFactory List not initialized");
				for (Method method : annotatedMethods.keySet()) {
					for (EventListenerFactory factory : factories) {
						if (factory.supportsMethod(method)) {
							Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
							ApplicationListener<?> applicationListener =
									factory.createApplicationListener(beanName, targetType, methodToUse);
							if (applicationListener instanceof ApplicationListenerMethodAdapter) {
								((ApplicationListenerMethodAdapter) applicationListener).init(context, this.evaluator);
							}
							context.addApplicationListener(applicationListener);
							break;
						}
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug(annotatedMethods.size() + " @EventListener methods processed on bean '" +
							beanName + "': " + annotatedMethods);
				}
			}
		}
	}
```



由方法生成`Listener`的逻辑由`EventListenerFactory`完成的，这又分为两种，一种是普通的`@EventLintener`  另一种是`@TransactionalEventListener` ，是由两个工厂处理的。



## 4.总结

​	上面介绍了`@EventListener`的原理，其实上面方法里还有一个`@TransactionalEventListener`注解，其实原理是一模一样的，只是这个监听者可以选择在事务完成后才会被执行，事务执行失败就不会被执行。

​	这两个注解的逻辑是一模一样的，并且`@TransactionalEventListener`本身就被标记有`@EventListener`，

只是最后生成监听器时所用的工厂不一样而已。

