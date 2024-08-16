---



layout:     post
title:      "springmvc异常处理"
subtitle:   ""
date:       2020-06-23
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- java
- 全局异常处理
---



# Springmvc全局异常处理,实现原理



​	`Springmvc`的异常处理是由下面这个接口提供的，只有一个方法，用来处理异常。称之为`异常处理器`。

```java
public interface HandlerExceptionResolver {

	ModelAndView resolveException(
			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);

}

```



​		`DispatcherServlet`在初始化时，会调用下面这个方法，在这里初始化所有的异常处理器。这里的逻辑是如果容器内存在该类型的实例，就用容器内的，如果不存在就使用默认`spring.factories`文件里写的默认值。

​		并且可以选择是搜索所有`HandlerExceptionResolver.class`类型的`bean`，还是只搜索名字叫`handlerExceptionResolver`的`bean`，由字段`detectAllHandlerExceptionResolvers`决定，默认搜索全部的。

```java
	private void initHandlerExceptionResolvers(ApplicationContext context) {
		this.handlerExceptionResolvers = null;
			
        
		if (this.detectAllHandlerExceptionResolvers) {
			Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils
					.beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerExceptionResolvers = new ArrayList<HandlerExceptionResolver>(matchingBeans.values());
				// We keep HandlerExceptionResolvers in sorted order.
				OrderComparator.sort(this.handlerExceptionResolvers);
			}
		}
		else {
			try {
				HandlerExceptionResolver her =
						context.getBean(HANDLER_EXCEPTION_RESOLVER_BEAN_NAME, HandlerExceptionResolver.class);
				this.handlerExceptionResolvers = Collections.singletonList(her);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, no HandlerExceptionResolver is fine too.
			}
		}

		if (this.handlerExceptionResolvers == null) {
			this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
			
		}
	}
```





​		`doDispatch`方法里，将查找`controller`并调用。渲染结果会调用下面这个方法，用于处理结果返回值。其中参数`exception`不为空时表示有错误发生的，就用到了异常处理器。

```java
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;
		//异常不为空
		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);		
                //处理异常，返回处理结果
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// Did the handler return a view to render?
		if (mv != null && !mv.wasCleared()) {
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("No view rendering, null ModelAndView returned.");
			}
		}

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}

		if (mappedHandler != null) {
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}
```



​		多个异常处理器会以责任链模式调用，这里可以看出，如果配置多个异常处理器，通过返回值是否为`null`来判断是否处理完成了。

```java
@Nullable
	protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
			@Nullable Object handler, Exception ex) throws Exception {

		// Success and error responses may use different content types
		request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

		// 
		ModelAndView exMv = null;
		if (this.handlerExceptionResolvers != null) {
            //责任链模式
			for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
				exMv = resolver.resolveException(request, response, handler, ex);
				if (exMv != null) {
					break;
				}
			}
		}
		if (exMv != null) {
			if (exMv.isEmpty()) {
				request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
				return null;
			}
			// We might still need view name translation for a plain error model...
			if (!exMv.hasView()) {
				String defaultViewName = getDefaultViewName(request);
				if (defaultViewName != null) {
					exMv.setViewName(defaultViewName);
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using resolved error view: " + exMv, ex);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Using resolved error view: " + exMv);
			}
			WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
			return exMv;
		}

		throw ex;
	}
```





​		下面介绍`ExceptionHandlerExceptionResolver`源码，我们经常使用的就是这个，他需要配合

`RequestMappingHandlerMapping`，`RequestMappingHandlerAdapter`使用。这是我们最常用的组合。





​		处理异常的逻辑如下，这里会优先查找`controller`里面的`@ExceptionHandler`注解标注的方法，如果没有则使用全局`@ControllerAdvice`里面的`@ExceptionHandler`标记的全局处理函数。



```java
	@Override
	protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod, Exception exception) {
		//查找该异常类型对应的处理函数
		ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
		if (exceptionHandlerMethod == null) {
			return null;
		}

	//这里设置了参数处理器，和返回值处理器		exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);	exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		ModelAndViewContainer mavContainer = new ModelAndViewContainer();

		try {
			//开始回掉异常处理函数
			exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception);
		}
		catch (Exception invocationEx) {
			logger.error("Failed to invoke @ExceptionHandler method: " + exceptionHandlerMethod, invocationEx);
			return null;
		}

		if (mavContainer.isRequestHandled()) {
			return new ModelAndView();
		}
		else {
			ModelAndView mav = new ModelAndView().addAllObjects(mavContainer.getModel());
			mav.setViewName(mavContainer.getViewName());
			if (!mavContainer.isViewReference()) {
				mav.setView((View) mavContainer.getView());
			}
			return mav;
		}
	}



protected ServletInvocableHandlerMethod getExceptionHandlerMethod(HandlerMethod handlerMethod, Exception exception) {
    //优先解析类里面的处理函数
		if (handlerMethod != null) {
			Class<?> handlerType = handlerMethod.getBeanType();
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
    //其次查找全局的处理函数
		for (Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry : this.exceptionHandlerAdviceCache.entrySet()) {
			Method method = entry.getValue().resolveMethod(exception);
			if (method != null) {
				return new ServletInvocableHandlerMethod(entry.getKey().resolveBean(), method);
			}
		}
		return null;
	}

```

​	



​		还有一点，`@ExceptionHandler`注解能添加`exception`类型的`value`值，只有匹配的异常类型才会使用此函数处理。如果没有在注解上标识能处理那种异常，则会解析方法的参数，参数中存在哪种异常就处理哪种。所以下面两种写法是等同的。

​		如果同时存在，则以注解上的为准。

```java
	@ExceptionHandler
	public String exception(RuntimeException e) {
		return e.getMessage();
	}

	@ExceptionHandler(value = RuntimeException.class)
	public String exception() {
		return "";
	}
```



​		还有一种常用的用法，用于接口的异常处理中，返回值是需要转成`Json`形式的，这里要在处理器上添加`@ResponseBody`注解，并且返回值是一个可以序列化成`Json`的对象。

```java
	@ExceptionHandler
	@ResponseBody
	public JsonResource exception(Exception e) {
		return JsonResource.ofFail(e.toString());
	}
	
```



​		这种情况下需要配置返回值处理器，查看`ExceptionHandlerExceptionResolver`内的代码，其在初始化方法里初始化了`returnValueHandlers`。

```java
	public void afterPropertiesSet() {
		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
		initExceptionHandlerAdviceCache();
	}


	protected List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<HandlerMethodReturnValueHandler>();

		// Single-purpose return value types
		handlers.add(new ModelAndViewMethodReturnValueHandler());
		handlers.add(new ModelMethodProcessor());
		handlers.add(new ViewMethodReturnValueHandler());
		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.contentNegotiationManager));

		// Annotation-based return value types
		handlers.add(new ModelAttributeMethodProcessor(false));
		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.contentNegotiationManager));

		// Multi-purpose return value types
		handlers.add(new ViewNameMethodReturnValueHandler());
		handlers.add(new MapMethodProcessor());

		// Custom return value types
		if (getCustomReturnValueHandlers() != null) {
			handlers.addAll(getCustomReturnValueHandlers());
		}

		// Catch-all
		handlers.add(new ModelAttributeMethodProcessor(true));

		return handlers;
	}
```



​		`returnValueHandlers`不是本文要将的，这里说明一点`RequestResponseBodyMethodProcessor`是用来处理带有`@ResponseBody`注解的方法返回值，它的构造方法是要传入`MessageConverters`，要想将对象转成Json，需要给他配置一个`MappingJackson2HttpMessageConverter`才可以。

​	

​		如果是`xml`方式配置，就是下面这样。。

```xml
<bean class="org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver">
        <property name="messageConverters">
            <list>
                <ref bean="stringHttpMessageConverter"/>
                <ref bean="jacksonHttpMessageConverter"/>
            </list>
        </property>
    </bean>
```

​		

​		使用`java`代码方式也是同理的，就是要将能处理返回值的`messageConverter`加入到`messageConverter`列表里。这里是继承它，在构造方法里加入`messageConverter`即可

```java
@Component
public class HandlerExceptionResolvers extends ExceptionHandlerExceptionResolver {

    public HandlerExceptionResolvers() {
        getMessageConverters().add(new MappingJackson2HttpMessageConverter());
    }
}

```





## 总结

#### 一般组合

​		一般是要使用`ExceptionHandlerExceptionResolver` + `RequestMappingHandlerMapping` + `RequestMappingHandlerAdapter`这一组组合来处理请求的，`spring4.x`版本默认也是这样。



#### 配置方式

​		通过在方法上添加`@ExceptionHandler`注解可以将方法标记为一个异常处理器，并且在`Controller`里面写的异常处理器优先级大于`@ControllerAdvice`类里写的。

​	`@ExceptionHandler`注解上或者方法参数上都可以指定能够处理的异常类型，一般写在参数里就可以。



#### 注意事项

​		异常处理的返回值会使用返回值处理器做处理，所以也需要配置返回值处理器，和配置`RequestMappingHandlerAdapter`是一样的。通用做法就是注入`MappingJackson2HttpMessageConverter`进去。

​		





