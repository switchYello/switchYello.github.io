---
layout:     post
title:      "spring中ObjectFactory 和 ObjectProvider"
subtitle:   ""
date:       2022-03-26
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring
- ObjectFactory
- ObjectProvider
typora-root-url: ..
---


# spring中ObjectFactory 和 ObjectProvider


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





