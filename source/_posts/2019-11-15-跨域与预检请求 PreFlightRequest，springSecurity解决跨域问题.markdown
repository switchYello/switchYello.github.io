---
layout:     post
title:      "跨域与预检请求 PreFlightRequest，springSecurity解决跨域问题"
subtitle:   ""
date:       2019-11-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springSecurity
---


## 跨域与预检请求 PreFlightRequest，springSecurity解决跨域问题

* TOC
{:toc}

#### 什么是预检请求

当浏览器发送post请求时，一般请求体都比较大，但如果是跨域的话，服务器会拒绝该请求，传输的数据被丢弃。
为了不浪费流量，浏览器在发送post前先发送一个小的options请求，将接下来的请求方式等设置到请求头中，
服务器检查请求头，如果服务器允许这样的请求
则返回200并在相应头中设置以下消息给客户端，如允许什么样的请求头，请求方式，是否允许带cookie等，
否则就拒绝请求。
这样的话，浏览器就不会再发送接下来的post请求了，从而节省资源。


- 其中两个请求头就是
	- `Access-Control-Request-Method:POST`,表示稍后的请求使用什么method进行请求，这里是 port，
	
	- `Access-Control-Request-Headers:content-type`,表示稍后的请求中自定义请求头是什么这里我是 content-type

	

#### 查看spring的CorsFilter源码

	在servlet环境下，请求都要经过filter，spring为我们提供了CorsFilter来处理这个问题。

```java
	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
			FilterChain filterChain) throws ServletException, IOException {
		//必须带有origin头部的请求，不带的则直接放行
		if (CorsUtils.isCorsRequest(request)) {
			CorsConfiguration corsConfiguration = this.configSource.getCorsConfiguration(request);
			if (corsConfiguration != null) {
				//验证是否有效，源码在下面
				boolean isValid = this.processor.processRequest(corsConfiguration, request, response);
				//如果不是有效的||是预检请求则直接返回，表示拒绝
				//这里如果验证失败了，response中会设置跨域提示相关的头部，则直接return告诉浏览器
				//如果验证成功了，但是是预检请求，则response中设置了同意的头部，直接返回即可
				if (!isValid || CorsUtils.isPreFlightRequest(request)) {
					return;
				}
			}
		}

		filterChain.doFilter(request, response);
	}

```

	上面源码可以看出，拦截器对请求进行检查，对是预检请求的request进行处理
	检查没通过则isValid为false，直接返回，此时response被设置了401状态码，前台看到的是401
	如果检查通过，但是是预检请求也直接返回，此时前台收到的是无内容的200响应

#### 如何判断是预检请求的逻辑

```java

	public static boolean isPreFlightRequest(HttpServletRequest request) {
		//
		return (isCorsRequest(request) 
				&& HttpMethod.OPTIONS.matches(request.getMethod()) 
				&&request.getHeader(HttpHeaders.ACCESS_CONTROL_REQUEST_METHOD) != null);
	}


```

	上面源码isPreFlightRequest中三个`&&`连接的判断可以看出，满足预检请求的条件是
	1.是cors请求，即带有origin头的，如get方式的请求就不带有origin头，所以就不是cors请求
	2.必须是options的请求  
	3.带有Access-Control-Request-Method请求头，表示接下来的请求方式
	满足上面三个条件就是预检请求了

#### this.processor.processRequest(corsConfiguration, request, response)这里面做了什么


```java
public boolean processRequest(@Nullable CorsConfiguration config, HttpServletRequest request,
			HttpServletResponse response) throws IOException {
		//非cors请求的直接返回true
		if (!CorsUtils.isCorsRequest(request)) {
			return true;
		}
		//设过Access-Control-Allow-Origin的直接返回true，如果此filter前面还有其他filter处理过了，就会有Access-Control-Allow-Origin头了
		ServletServerHttpResponse serverResponse = new ServletServerHttpResponse(response);
		if (responseHasCors(serverResponse)) {
			logger.trace("Skip: response already contains \"Access-Control-Allow-Origin\"");
			return true;
		}
		//是同源的直接返回true，同源的请求满足cors，允许放行
		ServletServerHttpRequest serverRequest = new ServletServerHttpRequest(request);
		if (WebUtils.isSameOrigin(serverRequest)) {
			logger.trace("Skip: request is from same origin");
			return true;
		}
		//得到是否是预检请求
		boolean preFlightRequest = CorsUtils.isPreFlightRequest(request);
		//如果config为空，则没有配置文件，然后判断如果是预检请求则拒绝，如果不是预检请求则放行。
		if (config == null) {
			if (preFlightRequest) {
				rejectRequest(serverResponse);
				return false;
			}
			else {
				return true;
			}
		}
		//真实检测逻辑，获取各种请求头和配置文件进行对比
		return handleInternal(serverRequest, serverResponse, config, preFlightRequest);
	}



protected boolean handleInternal(ServerHttpRequest request, ServerHttpResponse response,
			CorsConfiguration config, boolean preFlightRequest) throws IOException {
		//获取origin头
		String requestOrigin = request.getHeaders().getOrigin();
		//与配置的AllowedOrigins进行比较，得到匹配的
		String allowOrigin = checkOrigin(config, requestOrigin);
		HttpHeaders responseHeaders = response.getHeaders();

		responseHeaders.addAll(HttpHeaders.VARY, Arrays.asList(HttpHeaders.ORIGIN,
				HttpHeaders.ACCESS_CONTROL_REQUEST_METHOD, HttpHeaders.ACCESS_CONTROL_REQUEST_HEADERS));
		//如果origin都不匹配，返回false
		if (allowOrigin == null) {
			logger.debug("Reject: '" + requestOrigin + "' origin is not allowed");
			rejectRequest(response);
			return false;
		}
		//获取请求method，如果是预检请求则获取Access-Control-Request-Method里的method
		HttpMethod requestMethod = getMethodToUse(request, preFlightRequest);
		//检查请求头是否是AllowedMethods里配置的
		List<HttpMethod> allowMethods = checkMethods(config, requestMethod);
		if (allowMethods == null) {
			logger.debug("Reject: HTTP '" + requestMethod + "' is not allowed");
			rejectRequest(response);
			return false;
		}
		//获取请求header，如果是预检请求则获取Access-Control-Request-Headers里配置的
		List<String> requestHeaders = getHeadersToUse(request, preFlightRequest);
		//请求头与AllowedHeader里的进行匹配
		List<String> allowHeaders = checkHeaders(config, requestHeaders);
		if (preFlightRequest && allowHeaders == null) {
			logger.debug("Reject: headers '" + requestHeaders + "' are not allowed");
			rejectRequest(response);
			return false;
		}
		//上面的method，origin，head 三个检测都通过的话，则说明请求是自己人请求的，放行设置响应头
		responseHeaders.setAccessControlAllowOrigin(allowOrigin);

		if (preFlightRequest) {
			responseHeaders.setAccessControlAllowMethods(allowMethods);
		}

		if (preFlightRequest && !allowHeaders.isEmpty()) {
			responseHeaders.setAccessControlAllowHeaders(allowHeaders);
		}

		if (!CollectionUtils.isEmpty(config.getExposedHeaders())) {
			responseHeaders.setAccessControlExposeHeaders(config.getExposedHeaders());
		}

		if (Boolean.TRUE.equals(config.getAllowCredentials())) {
			responseHeaders.setAccessControlAllowCredentials(true);
		}

		if (preFlightRequest && config.getMaxAge() != null) {
			responseHeaders.setAccessControlMaxAge(config.getMaxAge());
		}

		response.flush();
		return true;
	}

```


#### 总结，什么时候允许request向下进入servlet

- 非cors请求，即头部不带origin的：有效
- response上已经设置过Access-Control-Allow-Origin：有效
- 同源请求：有效
- 缺少cors配置，是预检请求：拒绝
- 缺少cors配置，不是预检请求：有效

- 如果头部中的origin和配置的AllowedOrigins不匹配：拒绝
- 如果头部中的Access-Control-Request-Method和配置的AllowedMethods不匹配：拒绝
- 如果头部中的Access-Control-Request-Headers与配置中AllowedHeader不匹配且是预检请求：拒绝



#### 在springSecurity 的config中配置cors的例子

```java
      http.cors().configurationSource(request -> {
            CorsConfiguration c = new CorsConfiguration();
			//设置允许的头部，与请求头中的origin进行比对
            c.setAllowedOrigins(Arrays.asList("http://192.168.31.114", "*"));
			//设置允许的method，和请求method或预检请求Access-Control-Request-Method指定的method进行比对
            c.setAllowedMethods(Arrays.asList("POST", "GET", "OPTIONS", "PUT"));
			//设置请求头，与请求head比对或预检请求的Access-Control-Request-Headers比对
            c.addAllowedHeader("content-type");
			//允许携带cookie
            c.setAllowCredentials(true);
            c.setMaxAge(1800L);
            return c;
        });


```
