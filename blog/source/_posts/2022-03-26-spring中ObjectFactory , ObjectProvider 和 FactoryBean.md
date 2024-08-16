---
layout:     post
title:      "spring中ObjectFactory , ObjectProvider 和 FactoryBean"
subtitle:   ""
date:       2022-03-26
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring
- ObjectFactory
- ObjectProvider
- FactoryBean
typora-root-url: ..
---


# spring中ObjectFactory , ObjectProvider 和 FactoryBean


## ObjectFactory , ObjectProvider

这两个类是spring的提供的工厂方法的辅助类，
ObjectFactory在 1.0.2版本添加进去的，他只有一个getObject方法
ObjectProvider在4.3版本添加，它继承了ObjectFactory接口，添加了更多方法，如`getIfAvailable`，这种可选方法。

请看下面的相关源码，这里是解析自动注入依赖的方法，他会检测到依赖注入的是Optional,ObjectFactory,ObjectProvider,Provider进行特殊处理。


`org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveDependency`

```java
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
    if (Optional.class == descriptor.getDependencyType()) {
        return createOptionalDependency(descriptor, requestingBeanName);
    }
    else if (ObjectFactory.class == descriptor.getDependencyType() ||
            ObjectProvider.class == descriptor.getDependencyType()) {
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    }
    else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
    }
    else {
        Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
                descriptor, requestingBeanName);
        if (result == null) {
            result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
}
```

除了上面两个类之外，还可以使用 `javax.inject.Provider`类代替，但功能比较简单和ObjectFactory一致。
ObjectFactory , ObjectProvider主要在依赖注入时使用，如注入不一定存在的bean，或者注入非单例的bean，这样每次都能获取到新的对象。

如何使用这两个类呢，下面是一个示例。

```java
@Service
public class SchoolService {

    @Resource
    ObjectFactory<StudentService> studentServiceObjectFactory;
    
    @Resource
    ObjectProvider<StudentService> studentServiceObjectProvider;
    
    void doSomething(){
        
        //1. 使用ObjectFactory，从容器中获取bean
        StudentService object = studentServiceObjectFactory.getObject();
        

        //2. 使用ObjectProvider获取bean
        StudentService bean = studentServiceObjectProvider.getObject();
        
        //3. 使用ObjectProvider获取bean，如果不存在返回null
        StudentService ifAvailable = studentServiceObjectProvider.getIfAvailable();

    }
}

```


## FactoryBean
factoryBean与上面两个没有太大联系，他属于bean的生产者，有时我们要手工控制bean的创建过程，可以使用这个。如下示例：

该示例创建一个StudentServiceFactory，用于生产StudentService，默认生产的对象会被缓存。
如果其他类需要使用StudentService时，会先从容器中获取到该工厂，然后使用工厂生产一个StudentService。
再说一遍，生产的此对象默认会被缓存，下次再获取时就不会再生产了，可通过从写FactoryBean的方法控制是否单例模式。

```java
@Service
public class StudentServiceFactory implements FactoryBean<StudentService> {
    @Override
    public StudentService getObject() throws Exception {
        return new StudentService();
    }

    @Override
    public Class<?> getObjectType() {
        return StudentService.class;
    }
}
```

这个FactoryBean可以实现对象更复杂的创建过程，如根据接口动态代理生成一个Bean（没错Mybatis的mapper就是这个原理）。



## 总结

ObjectFactory , ObjectProvider 和 FactoryBean虽然名称相似，但是作用却不同。

ObjectFactory是初版提供的，用于实现非单例模式的依赖注入时多次获取不同对象。

ObjectProvider是4.x版本提供的，与ObjectFactory功能类似，但提供了更多方法，如`getIfAvailable()`这样的方法.

FactoryBean是控制复杂对象的创建用的，与上面两个并没有关系。
