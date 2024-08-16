---
layout:     post
title:      "RequestBodyAdvice ResponseBodyAdvice"
subtitle:   ""
date:       2021-01-07
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring
typora-root-url: ..
---

## RequestBodyAdvice ResponseBodyAdvice增强类


RequestBodyAdvice类和ResponseBodyAdvice是spring提供的接口，它可以在请求参数解析前，和响应输出前对controller的返回值
进行拦截，替换。

先看RequestBodyAdvice的Api，它可以在从流中读取参数前，读取参数后进行拦截
```java

public interface RequestBodyAdvice {

	boolean supports(MethodParameter methodParameter, Type targetType,
			Class<? extends HttpMessageConverter<?>> converterType);

	HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException;

	Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType);

	@Nullable
	Object handleEmptyBody(@Nullable Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType);

}
```

再看ResponseBodyAdvice的Api,它可以在输出返回值前，对响应进行拦截
```java
public interface ResponseBodyAdvice<T> {

	boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType);

	@Nullable
	T beforeBodyWrite(@Nullable T body, MethodParameter returnType, MediaType selectedContentType,
			Class<? extends HttpMessageConverter<?>> selectedConverterType,
			ServerHttpRequest request, ServerHttpResponse response);
}

```


他的加载是在下面这个方法里面实现的
>初始化方法
> org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#afterPropertiesSet
>被调用
>org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#initControllerAdviceCache

他在扫描@controllerAdvice类时，如果发现@controllerAdvice的类同时也是RequestBodyAdvice或ResponseBodyAdvice
则将其加入集合中。
```java
private void initControllerAdviceCache() {
		if (getApplicationContext() == null) {
			return;
		}

		List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());

		List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();

		for (ControllerAdviceBean adviceBean : adviceBeans) {
			Class<?> beanType = adviceBean.getBeanType();
			if (beanType == null) {
				throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
			}
			Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
			if (!attrMethods.isEmpty()) {
				this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
			}
			Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
			if (!binderMethods.isEmpty()) {
				this.initBinderAdviceCache.put(adviceBean, binderMethods);
			}
		
			if (RequestBodyAdvice.class.isAssignableFrom(beanType) || ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
				requestResponseBodyAdviceBeans.add(adviceBean);
			}
		}

		if (!requestResponseBodyAdviceBeans.isEmpty()) {
			this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
		}
	}

```


加入集合存储起来，并且在创建`RequestResponseBodyMethodProcessor`时，将其传递过去。
```java
private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>(20);
		// Single-purpose return value types
		handlers.add(new ModelAndViewMethodReturnValueHandler());
		handlers.add(new ModelMethodProcessor());
		handlers.add(new ViewMethodReturnValueHandler());
		handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters(),
				this.reactiveAdapterRegistry, this.taskExecutor, this.contentNegotiationManager));
		handlers.add(new StreamingResponseBodyReturnValueHandler());
		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));
		handlers.add(new HttpHeadersReturnValueHandler());
		handlers.add(new CallableMethodReturnValueHandler());
		handlers.add(new DeferredResultMethodReturnValueHandler());
		handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));

		// Annotation-based return value types
		handlers.add(new ModelAttributeMethodProcessor(false));
		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));

		// Multi-purpose return value types
		handlers.add(new ViewNameMethodReturnValueHandler());
		handlers.add(new MapMethodProcessor());

		// Custom return value types
		if (getCustomReturnValueHandlers() != null) {
			handlers.addAll(getCustomReturnValueHandlers());
		}

		// Catch-all
		if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
			handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
		}
		else {
			handlers.add(new ModelAttributeMethodProcessor(true));
		}

		return handlers;
	}
```

RequestResponseBodyMethodProcessor类就是处理ResponseBody响应的返回值处理器，他在将返回值转换成json返回前调用ResponseBodyAdvice
，我们可以在此修改或替换返回值的内容


同理RequestBodyAdvice是在RequestResponseBodyMethodProcessor解析参数时发挥作用的。



总结一下就是，
1. 这两个类一个是在解析参数时发挥作用，一个是在输出返回值前发挥作用
2. 这两个类的加载时机是在加载@controllerAdvice类时顺便判断的，所以要将这两个类上标注@controllerAdvice才能生效
3. 输出响应必须使用RequestResponseBodyMethodProcessor处理器或HttpEntityMethodProcessor处理器，这两个Advice才能生效

一般我用它输出请求响应的值记录到日志里，似乎也没有其他用处非得用这两个不可。



