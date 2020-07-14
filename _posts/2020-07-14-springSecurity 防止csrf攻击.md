---
layout:     post
title:      "springSecurity 防止csrf攻击"
subtitle:   ""
date:       2020-07-14
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- springSecurity
- csrf
typora-root-url: ..
---



# springSecurity 防止csrf攻击

​	

​		启用csrf filter。

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf();
}	
```



​	CsrfFilter处理方法

```java
protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain filterChain)
					throws ServletException, IOException {
		request.setAttribute(HttpServletResponse.class.getName(), response);
		
    	//从存储token容器内获取，默认是session中获取
		CsrfToken csrfToken = this.tokenRepository.loadToken(request);
		final boolean missingToken = csrfToken == null;
        //如果不存在则创建一个
		if (missingToken) {
			csrfToken = this.tokenRepository.generateToken(request);
			this.tokenRepository.saveToken(csrfToken, request, response);
		}
		request.setAttribute(CsrfToken.class.getName(), csrfToken);
		request.setAttribute(csrfToken.getParameterName(), csrfToken);
		
    
    	//这里判断某些请求不进行csrf过滤，默认将"GET", "HEAD", "TRACE", "OPTIONS"请求方式排除掉
    	//因为不建议使用上面四种方式做数据更新
		if (!this.requireCsrfProtectionMatcher.matches(request)) {
			filterChain.doFilter(request, response);
			return;
		}
		
    	//从head或paran里获取csrf token
		String actualToken = request.getHeader(csrfToken.getHeaderName());
		if (actualToken == null) {
			actualToken = request.getParameter(csrfToken.getParameterName());
		}
    	//比较请求上传的token与session存储的token是否一致
		if (!csrfToken.getToken().equals(actualToken)) {
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Invalid CSRF token found for "
						+ UrlUtils.buildFullRequestUrl(request));
			}
            //不一致或session里缺少token走拒绝分支，返回403啥的
			if (missingToken) {
				this.accessDeniedHandler.handle(request, response,
						new MissingCsrfTokenException(actualToken));
			}
			else {
				this.accessDeniedHandler.handle(request, response,
						new InvalidCsrfTokenException(csrfToken, actualToken));
			}
			return;
		}

		filterChain.doFilter(request, response);
	}

```



​			上面要求客户端在进行如 `post，put，delete`请求时，在参数或head里携带登录时生成的token。`get head`  等不拦截，要求服务端符合`Restful`规范，不要用get更新用户数据。



### 前端获取token

​		像上面那样简单的启用`csrf`，用户登录时会设置`token`到`session`里，那前端怎么获取呢？ 可以开一个接口返回此token，因为跨域原因，第三方网站是无法获取的，自己网站在打开页面时获取一下存储在页面上。



### Token有效期和服务重启问题

​	  	因为`token`存储在`session`里的，有效期就是`session`的有效期，在此期间如果服务器被强制重启`session`来不及钝化这将导致客户端使用正确的`token`，但无法验证通过，影响用户体验，可以将`token`存储在`redis`或数据库里，并设置有效期定期删除。



### 百度贴吧的例子

​		参考一下百度贴吧发帖请求时需要参数tbs，这个参数可以从接口 http://tieba.baidu.com/dc/common/tbs 上获取，每次请求都返回一个不一样的token，发帖时将携带token才能发帖成功。他的做法应该是每次调用接口都生成一个token，服务端保存最新的若干个token，发帖时只要和其中一个token一致即可验证通过。



### 其他方案

​		除了将 token 存储在`session`里外，还可以存储在`cookie`里。

​		用户登录成功后，生成`token`放入`cookie`里，客户端发起请求时将`cookie`里的`token`取出来放在参数上。服务端比较参数上的`token`和`cookie`里的`token`是否一致即可。因为第三方网站无法获取本站的`cookie`，也就无法伪造参数上的token，所以这种方案也是安全的。

​		



## springSecurity csrf的主要配置方法

```Java
  
  //设置session策略，登录成功后会回调此策略
  http.csrf().sessionAuthenticationStrategy();
  //设置token存储仓库，默认是存储在session里，可以设置存储在数据库里，或客户端cookie里      
  http.csrf().csrfTokenRepository();
  //需要忽略的路径
  http.csrf().ignoringAntMatchers();
  //强制验证的路径，即使是get也验证
  http.csrf().requireCsrfProtectionMatcher()
      
```



