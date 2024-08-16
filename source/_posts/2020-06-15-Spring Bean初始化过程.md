---



layout:     post
title:      "Spring Bean初始化过程"
subtitle:   ""
date:       2020-06-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- java
- spring
- Bean初始化过程
---



# Spring Bean初始化过程



​	本文讨论Spring容器下的Bean初始化过程。

创建SpringBoot项目，创建如下类，通过它研究Bean的初始化过程。

```java
@Component
public class Cat implements ApplicationContextAware {

    public Cat() {
        System.out.println("cat 被创建");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("aware");
    }
}
```



直接将断点定位到`org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons`方法上，执行到这里容器已经将Bean全部解析出来，然后逐个初始化，我们直接查看初始化过程。



​	可以看出，他是遍历beanNames列表，逐个初始化，遍历有两个分支，下面的分支是调用`getBean(beanName)`方法

```java
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}

```



​	继续按照断点走，最后来到这里。这里要做的是,使用反射创建Bean，对Bean进行初始化。

详细解释看注释

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		BeanWrapper instanceWrapper = null;
            //解析出beanName的类型，并通过反射new 出来 
    		//这里new出来的只是半成品，还没有注入属性
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
    	//将此半成品添加到列表里，处理循环依赖
    	//假设此类是A，如果依赖B，将会去创建B，但B又依赖A，则会将这个半成品的A给B，这样B就算构造完成了（虽然给他的A是半成品），然后继续构造A，就能解决循环依赖的问题。
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		
		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
            //在这里进行依赖注入
			populateBean(beanName, mbd, instanceWrapper);
            //这里是对bean的初始化，回掉各种aware
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			
		}
		return exposedObject;
	}

```





下面是初始化Bean的过程

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
	
    	//回掉BeanNameAware，BeanClassLoaderAware，BeanFactoryAware
		invokeAwareMethods(beanName, bean);
		
		Object wrappedBean = bean;
		
       //使用ApplicationContextAwareProcessor，回掉
       //EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware，ApplicationEventPublisherAware，MessageSourceAware，ApplicationContextAware
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		
		//回掉初始化方法
		invokeInitMethods(beanName, wrappedBean, mbd);
		
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		

		return wrappedBean;
	}
```



这样一个正常的Bean就创建完成了，这个Bean经历了

​	反射创建

​	依赖注入 （通过InstantiationAwareBeanPostProcessor实现的）

​	回掉各种Aware

​	回掉Init方法





## 具有循环依赖的Bean的创建过程



在Cat类里，注入People类，同样的创建一个People类，注入Cat类。People类省略了

```java
@Component
public class Cat  {

    public Cat() {
        System.out.println("cat 被创建");
    }

    private People people;
	
    @Autowired
    public void setPeople(People people) {
        this.people = people;
    }

}
```



再看`org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor`

的下面方法

```java
	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
        //发现需要注入的方法或字段
		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
		}
		return pvs;
	}
```



```java
	public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
		Collection<InjectedElement> checkedElements = this.checkedElements;
		Collection<InjectedElement> elementsToIterate =
				(checkedElements != null ? checkedElements : this.injectedElements);
		if (!elementsToIterate.isEmpty()) {
			for (InjectedElement element : elementsToIterate) {
				element.inject(target, beanName, pvs);
			}
		}
	}
```



​		注入的过程就是找到需要注入的方法或字段，如果是方法解析出对应的参数，让后尝试从工厂里获取参数对应的bean，如果工厂里没有，工厂就会自己创建。



​		上面的示例将会走如下逻辑

​		先初始化Cat ->需要注入Peoper但不存在 -> 创建People->People需要Cat进行注入向工厂要->工厂内已经有了Cat的实例，只是尚未初始化完成是个半成品，返回此半成品->people初始化完成 ->Cat初始化完成。

```java
	@Override
		protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
			if (checkPropertySkipping(pvs)) {
				return;
			}
			Method method = (Method) this.member;
			Object[] arguments;
			if (this.cached) {
				// Shortcut for avoiding synchronization...
				arguments = resolveCachedArguments(beanName);
			}
			else {
				Class<?>[] paramTypes = method.getParameterTypes();
				arguments = new Object[paramTypes.length];
				DependencyDescriptor[] descriptors = new DependencyDescriptor[paramTypes.length];
				Set<String> autowiredBeans = new LinkedHashSet<>(paramTypes.length);
				Assert.state(beanFactory != null, "No BeanFactory available");
				TypeConverter typeConverter = beanFactory.getTypeConverter();
				for (int i = 0; i < arguments.length; i++) {
					MethodParameter methodParam = new MethodParameter(method, i);
					DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
					currDesc.setContainingClass(bean.getClass());
					descriptors[i] = currDesc;
					try {
                        //这里解析出参数，并尝试从工厂中获取参数
                        //如果工厂中不存在，则工厂会自己创建
						Object arg = beanFactory.resolveDependency(currDesc, beanName, autowiredBeans, typeConverter);
						if (arg == null && !this.required) {
							arguments = null;
							break;
						}
						arguments[i] = arg;
					}
					catch (BeansException ex) {
						throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(methodParam), ex);
					}
				}
				synchronized (this) {
					if (!this.cached) {
						if (arguments != null) {
							Object[] cachedMethodArguments = new Object[paramTypes.length];
							System.arraycopy(descriptors, 0, cachedMethodArguments, 0, arguments.length);
							registerDependentBeans(beanName, autowiredBeans);
							if (autowiredBeans.size() == paramTypes.length) {
								Iterator<String> it = autowiredBeans.iterator();
								for (int i = 0; i < paramTypes.length; i++) {
									String autowiredBeanName = it.next();
									if (beanFactory.containsBean(autowiredBeanName) &&
											beanFactory.isTypeMatch(autowiredBeanName, paramTypes[i])) {
										cachedMethodArguments[i] = new ShortcutDependencyDescriptor(
												descriptors[i], autowiredBeanName, paramTypes[i]);
									}
								}
							}
							this.cachedMethodArguments = cachedMethodArguments;
						}
						else {
							this.cachedMethodArguments = null;
						}
						this.cached = true;
					}
				}
			}
			if (arguments != null) {
				try {
					ReflectionUtils.makeAccessible(method);
                    //使用解析出来的参数，回掉方法
					method.invoke(bean, arguments);
				}
				catch (InvocationTargetException ex) {
					throw ex.getTargetException();
				}
			}
		}

```



​	

 查看doGetBean方法里的getSingleton(beanName)调用

```java
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
```

​	



此方法将尝试获取尚未构建完成的Bean

```java
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		//从单例实例map里获取
        Object singletonObject = this.singletonObjects.get(beanName);
		//没获取到，且当前正在循环依赖的列表里
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				//从earlySingletonObjects列表里获取，没取不到则从singletonFactory中创建
                singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```





什么时候将尚未初始化的Bean放入map里的呢？

​	原来是创建完成后，尚未初始化前放入的。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		instanceWrapper = createBeanInstance(beanName, mbd, args);
		
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		//在这里半成品的Bean放入singletonFactories里
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	

	
		return exposedObject;
	}

```



