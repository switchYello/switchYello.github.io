---
layout:     post
title:      "spring @lazy注解的使用"
subtitle:   ""
date:       2022-05-11
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring
- springboot
- 技巧
typora-root-url: ..
---


# spring  @lazy注解的使用



在spring中@lazy注解表达延迟的含义，但在不同情况下，这里的延迟并不是同一层意思。下面将描述我发现的两层含义。



## 1. 延迟初始化bean

​		首先我们知道，容器启动之前会扫描所有的class文件，并将需要加载到容器中的类，整理成BeanDefinition存储。容器启动时将依次将BeanDefinition构建成bean，构建过程中同时解决依赖注入和循环引用的问题。



​		但并不是所有的BeanDefinition都会被构建成bean，观察源码中下面方法 `org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons`

​		此为容器启动过程时，从BeanDefinition构建成bean的入口。这里有三种情况不会进行初始化，

  1. 非抽象类 

  2. 是单例模式的bean

  3. 非Lazy模式的

     **所以这里就体现了Lazy的第一层含义，添加Lazy的注解的bean不会在容器启动时主动创建**。



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
                    FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged(
                            (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
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
```





有两种方式添加Lazy注解，一种直接在类上加@Lazy注解。一种是如果使用@Bean模式创建的bean，在方法上添加@Lazy注解。



​		但这里Lazy的含义仅仅是不在容器启动时主动生成bean，但可能会被动生成bean。如果bean A被 bean B依赖，A是lazy的，在启动时虽然A不会被主动创建，但在创建B时，需要依赖A，此时A就会被动的创建。

​		所以仅对某个bean的创建添加Lazy意义不是特别大，因为bean相互之间都是有依赖关系的，即使不主动创建也会被被动创建！



## 2. 延迟依赖注入

第二种Lazy的方式就比较有用了，如下面的例子。

```java
    @Component
    public class A{

    }

    public class B{   
        @Lazy
        @Resource
        private A a;
    }
```



​		这个例子中B依赖A。在构建B时需要将A注入，但是我们添加了@Lazy注解，注入时并不会真的从容器中查找A，而是注入一个A的动态代理。在运行阶段，调用动态代理类的方法时，才会真的从容器中查找A。

​		下面代码是`org.springframework.context.annotation.CommonAnnotationBeanPostProcessor`

的内部类，这里是负责依赖注入时查找依赖的部分，`getResourceToInject`方法里进行判断，如我上面讲的，如果是lazy的依赖则会生成动态代理，否则才会从容器中真实获取。

​		`lazyLookup`变量的在`ResourceElement`的构造方法里获取的。

```java
private class ResourceElement extends LookupElement {

    private final boolean lazyLookup;

    public ResourceElement(Member member, AnnotatedElement ae, @Nullable PropertyDescriptor pd) {
        super(member, pd);
        Resource resource = ae.getAnnotation(Resource.class);
        String resourceName = resource.name();
        Class<?> resourceType = resource.type();
        this.isDefaultName = !StringUtils.hasLength(resourceName);
        if (this.isDefaultName) {
            resourceName = this.member.getName();
            if (this.member instanceof Method && resourceName.startsWith("set") && resourceName.length() > 3) {
                resourceName = Introspector.decapitalize(resourceName.substring(3));
            }
        }
        else if (embeddedValueResolver != null) {
            resourceName = embeddedValueResolver.resolveStringValue(resourceName);
        }
        if (Object.class != resourceType) {
            checkResourceType(resourceType);
        }
        else {
            // No resource type specified... check field/method.
            resourceType = getResourceType();
        }
        this.name = (resourceName != null ? resourceName : "");
        this.lookupType = resourceType;
        String lookupValue = resource.lookup();
        this.mappedName = (StringUtils.hasLength(lookupValue) ? lookupValue : resource.mappedName());
        Lazy lazy = ae.getAnnotation(Lazy.class);
        this.lazyLookup = (lazy != null && lazy.value());
    }

    @Override
    protected Object getResourceToInject(Object target, @Nullable String requestingBeanName) {
        return (this.lazyLookup ? buildLazyResourceProxy(this, requestingBeanName) :
                getResource(this, requestingBeanName));
    }
}
```





## 3.配合使用

综上所述，如果将@Lazy 添加在类上，虽不会主动创建，但被依赖时还会被创建。

如果将@Lazy添加到注入的字段上，可以推迟注入的时间到运行时，但依赖已经被创建了，只是没注入而已，该耗费的时间也已经耗费了。

将两者配合使用，即可达到运行时再创建需要的对象，如果不需要可一直不创建。


