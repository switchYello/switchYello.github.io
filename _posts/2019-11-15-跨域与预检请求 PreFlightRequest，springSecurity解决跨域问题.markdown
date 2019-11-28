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


### 跨域与预检请求 PreFlightRequest，springSecurity解决跨域问题

#### 什么事预检请求

	在浏览器执行请求前先执行一个options请求，并携带两个请求头
	一个是Access-Control-Request-Method:POST
	一个是Access-Control-Request-Headers:content-type

	第一个请求头表示稍后的请求使用什么method进行请求，这里是 port，
	第二个请求头表示稍后的请求中自定义请求头是什么这里我是 content-type


#### 查看spring的CorsFilter源码

```
	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
			FilterChain filterChain) throws ServletException, IOException {
		//必须带有origin头部的请求
		if (CorsUtils.isCorsRequest(request)) {
			CorsConfiguration corsConfiguration = this.configSource.getCorsConfiguration(request);
			if (corsConfiguration != null) {
				//验证通过
				boolean isValid = this.processor.processRequest(corsConfiguration, request, response);
				//没通过检测或者是预检请求则直接返回
				if (!isValid || CorsUtils.isPreFlightRequest(request)) {
					return;
				}
			}
		}

		filterChain.doFilter(request, response);
	}

```

	上面源码可以看出，拦截器对请求进行检查
	检查没通过则isValid为false，直接返回，此时response被设置了401状态码，前台看到的是401
	如果检查通过，但是是预检请求也直接返回，此时前台收到的是无内容的200响应

#### 如何判断是预检请求呢

```

	public static boolean isPreFlightRequest(HttpServletRequest request) {
		return (isCorsRequest(request) && HttpMethod.OPTIONS.matches(request.getMethod()) &&
				request.getHeader(HttpHeaders.ACCESS_CONTROL_REQUEST_METHOD) != null);
	}


```

	上面源码可以看出，满足预检请求的条件是
	1.是cors请求即带有origin头的
	2.必须是options的请求
	3.带有Access-Control-Request-Method请求头
	满足上面三个条件就是预检请求了

#### this.processor.processRequest(corsConfiguration, request, response)这里面做了什么

```
public boolean processRequest(@Nullable CorsConfiguration config, HttpServletRequest request,
			HttpServletResponse response) throws IOException {
		//非cors请求的直接返回true
		if (!CorsUtils.isCorsRequest(request)) {
			return true;
		}
		//设过Access-Control-Allow-Origin的直接返回true'
		ServletServerHttpResponse serverResponse = new ServletServerHttpResponse(response);
		if (responseHasCors(serverResponse)) {
			logger.trace("Skip: response already contains \"Access-Control-Allow-Origin\"");
			return true;
		}
		//是同源的直接返回true
		ServletServerHttpRequest serverRequest = new ServletServerHttpRequest(request);
		if (WebUtils.isSameOrigin(serverRequest)) {
			logger.trace("Skip: request is from same origin");
			return true;
		}
		//得到是否是预检请求
		boolean preFlightRequest = CorsUtils.isPreFlightRequest(request);
		if (config == null) {
			if (preFlightRequest) {
				rejectRequest(serverResponse);
				return false;
			}
			else {
				return true;
			}
		}
		//真实检测
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


#### 在springSecurity 的config中配置cors的例子

```

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