---
layout:     post
title:      "spring-dubbo启动过程"
subtitle:   ""
date:       2021-02-09
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- dubbo
typora-root-url: ..
---



# spring-dubbo启动过程



1. 首先@EnableDubbo注解将DubboComponentScanRegistrar引入进来

  		2. DubboComponentScanRegistrar类会注册一个ServiceAnnotationBeanPostProcessor，这是一个BeanDefinitionRegistry的后置处理器。
  		3. 后置处理器对BeanDefinitionRegistry进行后置处理，他会使用scan扫描EnableDubbo注解上标注的包名内的类，扫描到的同时会注册进spring。

**org.apache.dubbo.config.spring.beans.factory.annotation.ServiceAnnotationBeanPostProcessor#registerServiceBeans**

```java
 private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);

        scanner.setBeanNameGenerator(beanNameGenerator);
		//扫描带有注解的类，同时兼容阿里版本的dubbo
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
        scanner.addIncludeFilter(new AnnotationTypeFilter(com.alibaba.dubbo.config.annotation.Service.class));

        for (String packageToScan : packagesToScan) {

            // 扫描此包名下的类，此方法里扫描到后会直接注册到spring容器里
            scanner.scan(packageToScan);

            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
			//将扫描到的dubbo类包装成ServiceBean.class，也注册到spring容器
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }

            } else {
                if (logger.isWarnEnabled()) {
                    logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                            + packageToScan + "]");
                }
            }
        }
    }
```



4. 上述使用scan扫描到的bean，注册到spring的同时，会再包装成ServiceBean也注册到spring里面

5. ServiceBean实现了ApplicationListener接口，其会在监听到事件后，将接口暴漏出去

   ```java
       @Override
       public void onApplicationEvent(ContextRefreshedEvent event) {
           if (!isExported() && !isUnexported()) {
               if (logger.isInfoEnabled()) {
                   logger.info("The service ready on spring started. service: " + getInterface());
               }
               //将接口暴漏出去
               export();
           }
       }
   ```

   

## 注意

​		因为ServiceBean是ApplicationListener接口，spring初始化ApplicationListener时，会将ServiceBean一同初始化了，此时要求其内部的Dubbo类也要初始化，如果其没有初始化，ServiceBean在执行export时会因为找不到类而报错。



​		因为Dubbo的export是由spring容器的事件触发的，如果是单容器不会有问题，但是如果容器初始化时创建了子容器，而子容器的事件会发送到父容器里面的，导致父容器还没有初始化完成就export了。



