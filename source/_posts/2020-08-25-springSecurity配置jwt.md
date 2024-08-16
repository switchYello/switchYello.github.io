---
layout:     post
title:      "springSecurity配置jwt"
subtitle:   ""
date:       2020-08-25
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springSecurity
- spring
- jwt
typora-root-url: ..
---



# springSecurity配置jwt



​		在分布式项目下，同一个项目后端可能部署多次，通过负载均衡分配到每个实例上，传统的Session是每个实例独有的，在一个实例上登陆后，其他实例并不知道登录状态。要想解决此问题，有以下几种办法，下面进行分析。

## 前端部分

​		首先前端能存储数据的方式有两种，一种是前端通过`localstorage`主动存储数据，在发送请求时主动携带。另一种是后端将数据放入`Cookie`中，前端发起请求时浏览器自动携带。

对于后端来说，这两种方式并没有本质上的区别。



## 后端部分

​		后端能存储数据的方式有以下几种

1.  存储在共享介质中，如`mysql`，`redis`，分布式缓存，但本质上是一样的。项目仍然是有状态的，但是状态共享。

2. 存储在项目本身的缓存或`session`里。因为存储在自身的项目下，其他项目无法获取到数据。
3. 以`Cookie`的形式或者接口返回值的形式传给前端，前端请求时每次都带回来，这样其他项目就能拿到共享数据。



​			方案1 本质上的有状态的，只是状态是共享的。需要使用用户唯一标识来获取用户存储的数据，此标识可以存储在`Cookie`里也可以存储在`localStorage`由前台主动发送，这是对原始`cookie session`功能的改进，只是将`session`由单机变成分布式而已。

​		方案2 无法解决负载均衡下的多实例问题。

​		方案3 将信息下发送到前端，每个实例处理请求是拿到前端的信息，就能得到数据，不需要第三方存储，但是需要一种方案验证信息的有效性，且不宜过大。



## jwt以及变种

​		`jwt`就属于方案三，在登陆时将用户信息存储到前台，下次访问时在重新解析出信息就能判断用户是谁，权限是什么，是否登录。

​		标准的`jwt`规范是使用`json`存储数据。除了使用`json`外，也可以使用其他变种格式，比如用逗号分隔，都是可以的。	

​		`SpringSecurity`里自带了`RememberMeService`，这个原始用法是为了实现记住我功能，用户`session`关闭后下次访问能自动登录。如果用户压根没有`session`，那么每次登录都是自动登录，以此来验证用户身份也是可以的。

​		标准的`TokenBasedRememberMeServices`会将『用户名，过期时间，密码，盐』加一起进行md5，自动登录时进行同样的运算，如果获取的签名一致则允许自动登录，但是查询密码时仍然需要查询数据库，我们可以简化这一步，使他恒返回一个唯一的密码，因为有盐的存在，签名仍然是安全的，只不过用户改密码后`jwt`不会失效，不过`jwt`就是如此。

​		我们将生成的值放入cookie里，这样就不需要前台配合，仅后端就能实现。



## 配置过程

​	将`SpringSecurity`改为`STATELESS`模式，这样他就不会使用`session`了,

然后配置`RememberMeService`，我们继承`TokenBasedRememberMeServices`，重写其中的`retrievePassword`方法，并且设置一个`userDetailService`，传入构造方法里。

```java
public class BaseSecurityConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http){
		//配置为无状态模式,不使用session
	http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);  		 //配置form表单登录，以及登录后的执行逻辑
http.formLogin().loginPage("/error").loginProcessingUrl("/login").permitAll()
				.successHandler((request, response, authentication) -> {
					String str = "{\"s\":1,\"r\":\"login success\"}";
					sendOut(str, response);
				})
				.failureHandler((request, response, exception) -> {
					String str = "{\"s\":0,\"r\":\"" + exception.getMessage() + "\"}";
					sendOut(str, response);
				});
        
		//使用RememberMe功能实现jwt登录
		TokenBasedRememberMeServices rememberMeServices = new JwtRemember("tokenxxx");
		rememberMeServices.setAlwaysRemember(true);
		rememberMeServices.setCookieName("jwt.token");
		http.rememberMe().rememberMeServices(rememberMeServices).key("tokenxxx");
		
		//所有端口都需要验证
		http.authorizeRequests().anyRequest().authenticated();
	}


	private static class JwtRemember extends TokenBasedRememberMeServices {

		private static String salt = "_randow_salt";
        
		//传入userDetailService，返回的新用户的密码是 用户名 + salt
		protected JwtRemember(String key) {
			super(key, username -> new User(username, username + salt, Collections.emptyList()));
		}
        
		//重写解析密码的方法，密码固定返回 用户名 + salt
		@Override
		protected String retrievePassword(Authentication authentication) {
			return retrieveUserName(authentication) + salt;
		}

	}

	//配置全局userDetailsService 和 passwordEncoder
	@Override
	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder);
	}


	private UserDetailsService userDetailsService = username -> {
		User user = userService.getUserByUserName(username);
		if (user == null) {
			throw new UsernameNotFoundException(username);
		}
        return user;
	};


	private PasswordEncoder passwordEncoder = new PasswordEncoder() {
		@Override
		public String encode(CharSequence rawPassword) {
			return UserService.securityPassword(rawPassword.toString());
		}
		
		@Override
		public boolean matches(CharSequence rawPassword, String encodedPassword) {
			return encode(rawPassword).equals(encodedPassword);
		}
	};
}

```

​		初次登录时，会通过下面的`UserDetailsService` 和 `PasswordEncoder` 查询数据库，验证用户身份，登陆完成后`RememberMeService`会自动创建Cookie，Cookie的内容就是上面写的 。



​		第二次访问时，请求经过`RememberMeAuthenticationFilter`时，它检测到当前未登录，从Cookie中获取数据，自动登录。
这里是不查询数据库的。


下面是此`RememberService`进行生成`Token`的过程

```java

    public String makeTokenSignature(long tokenExpiryTime, String username, String password) {
        //用户名，过期时间，密码（此密码已被我们改为用户名+salt了）+ key
        String data = username + ":" + tokenExpiryTime + ":" + password + ":" + getKey();
        MessageDigest digest;
        try {
            digest = MessageDigest.getInstance("MD5");
        }
        catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("No MD5 algorithm available!");
        }
        return new String(Hex.encode(digest.digest(data.getBytes())));
    }

```





以下是自动登录的过程

```java

	@Override
	protected UserDetails processAutoLoginCookie(String[] cookieTokens,
			HttpServletRequest request, HttpServletResponse response) {
		
        //验证过期时间
		long tokenExpiryTime;
		try {
			tokenExpiryTime = new Long(cookieTokens[1]).longValue();
		}
		catch (NumberFormatException nfe) {
			throw new InvalidCookieException(
					"Cookie token[1] did not contain a valid number (contained '"
							+ cookieTokens[1] + "')");
		}

		if (isTokenExpired(tokenExpiryTime)) {
			throw new InvalidCookieException("Cookie token[1] has expired (expired on '"
					+ new Date(tokenExpiryTime) + "'; current time is '" + new Date()
					+ "')");
		}
        
        //通过用户名查找用户，这里使用的是构造方法里传入的userDetailSevice
		UserDetails userDetails = getUserDetailsService().loadUserByUsername(
				cookieTokens[0]);
        
        //使用查询到的userDetails 再次生成签名
		String expectedTokenSignature = makeTokenSignature(tokenExpiryTime,
				userDetails.getUsername(), userDetails.getPassword());
        
        //验证签名和cookie里携带的签名相同
		if (!equals(expectedTokenSignature, cookieTokens[2])) {
			throw new InvalidCookieException("Cookie token[2] contained signature '"
					+ cookieTokens[2] + "' but expected '" + expectedTokenSignature + "'");
		}
		return userDetails;
	}

```





下面是正常登录成功后，Cookie生成逻辑

```java
@Override
	public void onLoginSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication successfulAuthentication) {

		String username = retrieveUserName(successfulAuthentication);
        
        //此处的获取密码方法已被我们重写，返回 用户名+salt
		String password = retrievePassword(successfulAuthentication);
		
        //过期时间
		int tokenLifetime = calculateLoginLifetime(request, successfulAuthentication);
		long expiryTime = System.currentTimeMillis();
		expiryTime += 1000L * (tokenLifetime < 0 ? TWO_WEEKS_S : tokenLifetime);
		
        //生成签名
		String signatureValue = makeTokenSignature(expiryTime, username, password);
		//设置cookie
		setCookie(new String[] { username, Long.toString(expiryTime), signatureValue },tokenLifetime, request, response);

        
	}

```





## 总结

​		以上就是使用`SpringSecurity`做无状态服务的方法，除了第一次登陆时，后面验证是否登录是不需要查库的，我们也可以重写生成`Cookie`的方法，将更多的信息保存在前端。

​		`rememberMeService`是完全嵌入到`SpringSecurity`体系内的组件，且其中的设置`Cookie`，清理`Cookie`都已经有实现，我们用这个来实现`jwt`是相当方便的。

​		除此之外也可以使用自定义`filter`达到同样效果，但是利用`rememberMeService`实现起来比较简单。




