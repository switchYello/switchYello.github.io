---
layout:     post
title:      "springboot 如何正确配置springmvc"
subtitle:   ""
date:       2020-08-25
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring
- springmvc
- springboot
typora-root-url: ..
---



# springboot 如何正确配置springmvc



​		

​		虽然`Springboot`是开箱即用的，使用`Springmvc`也是十分简单，但是如何配置才是最好的呢？，看到很多人多它的用法不了解，本文讲一下它的用法。

将它之前，我们先了解下**什么是`Springmvc`**。

​	`Springmvc`是基于`servlet`体系的开发框架，他由`DispatcherServlet`拦截请求，根据请求路径使用`handlerMapper`查找到能执行请求的`handler`，再由`handlerAdapter`执行`handler`。

这其中又涉及到了如何映射?，如何解析参数?，如果处理返回值?，如果处理异常?，如果渲染页面?，等一系列问题，但是核心的操作就是下面三个。

1. `DispatcherServlet` 实现了servlet接口，负责拦截请求

2.  `handlerMapper` 负责路由

3. `handlerAdapter` 负责执行handler



所以`Springboot`的自动配置就是将上面的一系列类初始化好，再启动`Tomcat`，将`servlet`配置到`tomcat`里面。



## 1. WebMvcConfigurationSupport

​		打开`WebMvcConfigurationSupport`类查看代码，我么上面讲到的`Springmvc`需要的组件全部在此类中定义，并且留了大量的钩子函数，供我们自定义，我们可以继承此类，重写其中的钩子函数，想要修改那部分就找到其中的钩子函数，可以完成定制。

​		但是这种方式并不好，第一我们继承此类后，Springboot的自动配置就会失效，第二其他类如果想要定制Springmvc就不行了。



## 2.@EnableWebMvc注解

​		查看该注解的源码其引入了`DelegatingWebMvcConfiguration`类，这是一个代理类，并且继承`WebMvcConfigurationSupport`，他能自动查找`WebMvcConfigurer`，同样能对`Springmvc`进行定制，并且不止我们自己能能提供`WebMvcConfigurer`，其他框架也能提供`WebMvcConfigurer`，多个`WebMvcConfigurer`可以共存。所以比上面的方法更好。查看`WebMvcConfigurer`里面的方法，和`WebMvcConfigurationSupport`非常相似的。



## 3. 直接实现`WebMvcConfigurer`接口

​		查看`WebMvcAutoConfiguration`类，这是`Springboot`对`Springmvc`的自动配置类，其中有一个子类`EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration`，所以即使我么不使用`@EnableWebMvc`注解，也可以实现配置，并且此类中内置了大量有用的配置。

如默认资源路径配置，`OrderedHiddenHttpMethodFilter`，`OrderedFormContentFilter`等。

​		由于继承了`DelegatingWebMvcConfiguration`，所以我们想要自定义可以直接实现`WebMvcConfigurer`接口。



上面三种方式都可以实现自定义`Springmvc`，第一种方式是强烈不推荐的，如果不想要`WebMvcAutoConfiguration`类内默认配置的那些东西，可以使用第二种方式，如果想要默认配置的东西，可以使用第三种方式，建议使用第三种方法，因为默认配置有不少有用的东西。





## 总结

​		综上所述，配置`Springmvc`的最优雅方式就是实现`WebMvcConfigurer`接口，并让工厂扫描到，所有的自定义配置都通过钩子函数完成，不要使用`@EnableWebMvc`注解，更不要直接继承`WebMvcConfigurationSupport`类。

​		另外`SpringWebFlux`的配置思路和`Springmvc`基本一致，有兴趣学习webflux的可以顺着mvc的思路思考。

