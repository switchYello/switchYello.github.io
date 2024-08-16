---
layout:     post
title:      "ControllerAdvice参数的作用"
subtitle:   ""
date:       2021-01-04
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring
- controllerAdvice
typora-root-url: ..
---



# 本文讲一讲ControllerAdvice注解的讲解



首先看注解的定义

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface ControllerAdvice {

	@AliasFor("basePackages")
	String[] value() default {};

	@AliasFor("value")
	String[] basePackages() default {};

	Class<?>[] basePackageClasses() default {};

	Class<?>[] assignableTypes() default {};

	Class<? extends Annotation>[] annotations() default {};
}
```

​		首先，这个注解标记的类会被声明为全局异常处理器，但是它的优先级比controller中定义的ExceptionHandler注解要低。

​		注解内部有五个方法，其中value和basePackages是一样的。

1. `basePackages`表示限制包名，限制controller的类的包名，必须满足匹配
2. `basePackageClasses` 类型安全的限制包名方法，框架建议在想要限制的包名内创建一个空的类，这样比上面直接使用包名字符串要好
3. `assignableTypes `限制controller类的继承关系
4. `annotations` 限制controller上必须存在指定的注解

如果上面四个参数都没有填写，默认是匹配的。源码在下面。



ServletInvocableHandlerMethod.class

```java
	protected ServletInvocableHandlerMethod getExceptionHandlerMethod(HandlerMethod handlerMethod, Exception exception) {
		Class<?> handlerType = (handlerMethod != null ? handlerMethod.getBeanType() : null);

		if (handlerMethod != null) {
			ExceptionHandlerMethodResolver resolver = this.exceptionHandlerCache.get(handlerType);
			if (resolver == null) {
				resolver = new ExceptionHandlerMethodResolver(handlerType);
				this.exceptionHandlerCache.put(handlerType, resolver);
			}
			Method method = resolver.resolveMethod(exception);
			if (method != null) {
				return new ServletInvocableHandlerMethod(handlerMethod.getBean(), method);
			}
		}
 //上面如果找不到controller中定义的exceptionHandler，则使用全局异常处理器
		for (Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry : this.exceptionHandlerAdviceCache.entrySet()) {
//全局异常处理器必须满足条件    			
                if (entry.getKey().isApplicableToBeanType(handlerType)) {
				ExceptionHandlerMethodResolver resolver = entry.getValue();
				Method method = resolver.resolveMethod(exception);
				if (method != null) {
					return new ServletInvocableHandlerMethod(entry.getKey().resolveBean(), method);
				}
			}
		}
		return null;
	}

```



匹配逻辑源码

```java

//参数beanType就是controller对应的类
	public boolean isApplicableToBeanType(Class<?> beanType) {
//如果指定的三个限制都没有，默认是匹配的        
		if (!hasSelectors()) {
			return true;
		}
		else if (beanType != null) {
//限制包名
			for (String basePackage : this.basePackages) {
				if (beanType.getName().startsWith(basePackage)) {
					return true;
				}
			}
//限制类
			for (Class<?> clazz : this.assignableTypes) {
				if (ClassUtils.isAssignable(clazz, beanType)) {
					return true;
				}
			}
//限制注解
			for (Class<? extends Annotation> annotationClass : this.annotations) {
				if (AnnotationUtils.findAnnotation(beanType, annotationClass) != null) {
					return true;
				}
			}
		}
		return false;
	}

//检查注解上的三个参数是否存在
	private boolean hasSelectors() {
		return (!this.basePackages.isEmpty() || !this.assignableTypes.isEmpty() || !this.annotations.isEmpty());
	}

```





