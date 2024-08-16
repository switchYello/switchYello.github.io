---
layout:     post
title:      "springSecurity 退出登录"
subtitle:   ""
date:       2020-07-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- springSecurity
- logoutFilter
---



# springSecurity 退出登录

​	

​		这个`filter`比较简单，它要做的事情就是拦截退出登录的请求，执行退出逻辑，执行退出完成逻辑。



```java
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
		//匹配到是退出请求，默认是/logout
		if (requiresLogout(request, response)) {
			Authentication auth = SecurityContextHolder.getContext().getAuthentication();

			if (logger.isDebugEnabled()) {
				logger.debug("Logging out user '" + auth
						+ "' and transferring to logout destination");
			}
			//执行退出逻辑
			this.handler.logout(request, response, auth);
            //执行退出完成成功，回掉逻辑
			logoutSuccessHandler.onLogoutSuccess(request, response, auth);

			return;
		}

		chain.doFilter(request, response);
	}

```



## 修改默认退出路径

​	下面这样配置可以修改默认退出路径，原理是创建`LogoutFilter`时，将里面的匹配规则改为我们设置的路径。

```java
http.logout().logoutUrl("/my/logout")
```

​	 值得注意的是，如果开启csrf功能，则退出请求只支持post，否则可以支持四种请求方式。

```java
	private RequestMatcher getLogoutRequestMatcher(H http) {
		if (logoutRequestMatcher != null) {
			return logoutRequestMatcher;
		}
		if (http.getConfigurer(CsrfConfigurer.class) != null) {
			this.logoutRequestMatcher = new AntPathRequestMatcher(this.logoutUrl, "POST");
		}
		else {
			this.logoutRequestMatcher = new OrRequestMatcher(
				new AntPathRequestMatcher(this.logoutUrl, "GET"),
				new AntPathRequestMatcher(this.logoutUrl, "POST"),
				new AntPathRequestMatcher(this.logoutUrl, "PUT"),
				new AntPathRequestMatcher(this.logoutUrl, "DELETE")
			);
		}
		return this.logoutRequestMatcher;
	}
```



## 默认退出逻辑

​	在`LogoutConfigurer`类里可以看到，默认会添加两个退出逻辑。

​	首先删除session内的用户信息，再删除`SecurityContextHolder`保存的用户信息，这样就算退出完成了。

如果项目里用户信息是保存在数据库里的，可与添加自定义的`logoutHandler`。

```java
private LogoutFilter createLogoutFilter(H http) {
    	//这是是删除session，删除SecurityContextHolder
		logoutHandlers.add(contextLogoutHandler);
       //这个是发送事件的
		logoutHandlers.add(postProcess(new LogoutSuccessEventPublishingLogoutHandler()));
		LogoutHandler[] handlers = logoutHandlers.toArray(new LogoutHandler[0]);
		LogoutFilter result = new LogoutFilter(getLogoutSuccessHandler(), handlers);
		result.setLogoutRequestMatcher(getLogoutRequestMatcher(http));
		result = postProcess(result);
		return result;
	}
```



## 添加自定义logoutHandler

​		可以看到自己添加的`LogoutHandler`会添加到`logoutHandlers`这个List里。

```java
http.logout().addLogoutHandler(new LogoutHandler() {
    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
        System.out.println(authentication.getPrincipal() + "退出了");
    }
});

public LogoutConfigurer<H> addLogoutHandler(LogoutHandler logoutHandler) {
    this.logoutHandlers.add(logoutHandler);
    return this;
}
```



## 退出时自动删除Cookie

​	下面这样配置，可以在退出时自动删除指定的Cookie

```java
http.logout().deleteCookies("sessionId","usertoken");
```



​	原理就是添加了`CookieClearingLogoutHandler`，在退出时向`response`添加同名`cookie`，将客户端的覆盖掉。

```java
public LogoutConfigurer<H> deleteCookies(String... cookieNamesToClear) {
    return addLogoutHandler(new CookieClearingLogoutHandler(cookieNamesToClear));
}
```



## 退出成功执行逻辑



​	默认会重定向到`/login?logout`页面上，可以自定义重定向地址

```java
//自定义退出成功Url,会重定向到此url
http.logout().logoutSuccessUrl("/my/login");
```



​	重定向的原理就是用我们指定的路径创建一个`SimpleUrlLogoutSuccessHandler`。这个SuccessHandler调用时会做重定向操作。

```java
private LogoutSuccessHandler getLogoutSuccessHandler() {
    LogoutSuccessHandler handler = this.logoutSuccessHandler;
    if (handler == null) {
        handler = createDefaultSuccessHandler();
    }
    return handler;
}

private LogoutSuccessHandler createDefaultSuccessHandler() {
    SimpleUrlLogoutSuccessHandler urlLogoutHandler = new SimpleUrlLogoutSuccessHandler();
    urlLogoutHandler.setDefaultTargetUrl(logoutSuccessUrl);
    if (defaultLogoutSuccessHandlerMappings.isEmpty()) {
        return urlLogoutHandler;
    }
    DelegatingLogoutSuccessHandler successHandler = new DelegatingLogoutSuccessHandler(defaultLogoutSuccessHandlerMappings);
    successHandler.setDefaultLogoutSuccessHandler(urlLogoutHandler);
    return successHandler;
}
```



## 自定义 logoutSuccessHandler

​		有时候退出后不想重定向，而是数据给前台（前后端分离的项目应该都是这样）。可以自定义退出成功执行逻辑，可以覆盖上面的重定向操作。

​		下面这样就是退出成功后，输出字符串到前台。

```java
http.logout().logoutSuccessHandler(new LogoutSuccessHandler() {
    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        response.getWriter().write("logou success");
    }
});
```



## 多退出路径

退出后可以对页面进行重定向，也可以返回Json数据，有时候想要两者共存怎么办呢？

```
如访问 /api/logout，则返回json
访问 /page/logout，执行重定向到首页	
```

也是可以做到的。

​	首先自定义退出匹配规则，这样就能匹配多个退出路径了

```java
        //自定义退出匹配规则
        http.logout().logoutRequestMatcher(new OrRequestMatcher(
           new AntPathRequestMatcher("/api/logout"),
           new AntPathRequestMatcher("/page/logout")
        ));

```

​		为其中一个退出路径指定`logoutSuccessHandler`

```java
//指定匹配/api/logout的退出路径，退出成功后执行逻辑
http.logout().defaultLogoutSuccessHandlerFor(new LogoutSuccessHandler() {
    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        response.getWriter().write("log success");
    }
},new AntPathRequestMatcher("/api/logout"));
        
```

​		剩下的那个`/page/logout`没有指定处理逻辑，则会走默认的重定向到`logoutSuccessUrl`。

这样就实现了拥有两个退出方式，且返回方式不同。



## 所有配置

```java

        //1. 添加自定义退出路径
        http.logout().logoutUrl("/my/logout");
        //2. 添加自定义退出操作，如在这里修改数据库用户状态
        http.logout().addLogoutHandler(new LogoutHandler() {
            @Override
            public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
                System.out.println(authentication.getPrincipal() + "退出了");
            }
        });
        //3. 删除指定cookie
        http.logout().deleteCookies("sessionId", "usertoken");
        //4. 自定义退出成功重定向到Url
        http.logout().logoutSuccessUrl("/my/login");

        //5. 自定义退出成功Handler，此功能会覆盖4中的重定向操作
        http.logout().logoutSuccessHandler(new LogoutSuccessHandler() {
            @Override
            public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
                response.getWriter().write("logou success");
            }
        });

        //6.自定义退出匹配规则，可以存在多个。会覆盖1中的自定义路径
        http.logout().logoutRequestMatcher(new OrRequestMatcher(
                new AntPathRequestMatcher("/api/logout"),
                new AntPathRequestMatcher("/page/logout")
        ));
        //7. 指定匹配/api/logout的退出路径，退出成功后执行逻辑
			//满足匹配的走这里面的逻辑，不满足的走默认逻辑，默认逻辑是 4或者5
        http.logout().defaultLogoutSuccessHandlerFor(new LogoutSuccessHandler() {
            @Override
            public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
                response.getWriter().write("log success");
            }
        }, new AntPathRequestMatcher("/api/logout"));

        //8.是否删除session，删除ContentSecurityHolder内的认证
        http.logout().invalidateHttpSession(true);
        http.logout().clearAuthentication(true);
        //9.允许退出请求访问
        http.logout().permitAll();
        
```





## 总结

​		`SpringSecurity`的退出操作很简单，也很灵活。

































