---
layout:     post
title:      "springSecurity的两种rememberMe方式"
subtitle:   ""
date:       2019-10-30
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springSecurity
---


## springSecurity的两种rememberMe方式


	springSecurity 使用RememberMeAuthenticationFilter处理记住我功能，代码逻辑

```
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
		//当前处于未登录状态
		if (SecurityContextHolder.getContext().getAuthentication() == null) {
			//使用rememberMeServices进行自动登录，返回登录成功标识
			Authentication rememberMeAuth = rememberMeServices.autoLogin(request,response);

			if (rememberMeAuth != null) {

				try {
					//验证rememberMeAuth的key和rememberMeServices的key是否一致
					rememberMeAuth = authenticationManager.authenticate(rememberMeAuth);

					// Store to SecurityContextHolder
					SecurityContextHolder.getContext().setAuthentication(rememberMeAuth);

					onSuccessfulAuthentication(request, response, rememberMeAuth);

					if (successHandler != null) {
						successHandler.onAuthenticationSuccess(request, response,
								rememberMeAuth);

						return;
					}

				}
				catch (AuthenticationException authenticationException) {

			}

			chain.doFilter(request, response);
		}
		else {
			chain.doFilter(request, response);
		}
	}


```

### springSeciruity提供的实现 
	上面的filter使用RememberMeServices来处理自动登录，
	TokenBasedRememberMeServices和PersistentTokenBasedRememberMeServices是springSecurity提供的两种处理remenberMe的实现类
	他俩都继承自 AbstractRememberMeServices

### 0.AbstractRememberMeServices

```
	//自动登录方法
	public final Authentication autoLogin(HttpServletRequest request,
			HttpServletResponse response) {

		//获取cookie中的指定cookie
		String rememberMeCookie = extractRememberMeCookie(request);
		
		UserDetails user = null;

		try {
			//base64转成字符串，再按照.分隔成数组
			String[] cookieTokens = decodeCookie(rememberMeCookie);
			//子类实现的登录方法
			user = processAutoLoginCookie(cookieTokens, request, response);
			//检查用户状态，是否锁定，是否有效等
			userDetailsChecker.check(user);
			//返回验证成功的Authentication
			return createSuccessfulAuthentication(request, user);
		}
		catch (CookieTheftException cte) {
			cancelCookie(request, response);
			throw cte;
		}
		catch (UsernameNotFoundException noUser) {
			logger.debug("Remember-me login was valid but corresponding user not found.",
					noUser);
		}
		catch (InvalidCookieException invalidCookie) {
			logger.debug("Invalid remember-me cookie: " + invalidCookie.getMessage());
		}
		catch (AccountStatusException statusInvalid) {
			logger.debug("Invalid UserDetails: " + statusInvalid.getMessage());
		}
		catch (RememberMeAuthenticationException e) {
			logger.debug(e.getMessage());
		}

		cancelCookie(request, response);
		return null;
	}


```


### 1. 使用TokenBasedRememberMeServices如何实现processAutoLoginCookie方法的

```
protected UserDetails processAutoLoginCookie(String[] cookieTokens,
			HttpServletRequest request, HttpServletResponse response) {
		//验证cookie中解码完成的值数组长度是3，分别是[用户名，过期时间戳，md5(用户名：过期时间：密码：key)]
		if (cookieTokens.length != 3) {
			throw new InvalidCookieException("Cookie token did not contain 3"
					+ " tokens, but contained '" + Arrays.asList(cookieTokens) + "'");
		}

		long tokenExpiryTime;
		//数组第二个元素转成Long
		try {
			tokenExpiryTime = new Long(cookieTokens[1]).longValue();
		}
		catch (NumberFormatException nfe) {
			throw new InvalidCookieException(
					"Cookie token[1] did not contain a valid number (contained '"
							+ cookieTokens[1] + "')");
		}
		//验证是否过期
		if (isTokenExpired(tokenExpiryTime)) {
			throw new InvalidCookieException("Cookie token[1] has expired (expired on '"
					+ new Date(tokenExpiryTime) + "'; current time is '" + new Date()
					+ "')");
		}
		//使用UserDetailsService 查询出用户信息
		UserDetails userDetails = getUserDetailsService().loadUserByUsername(
				cookieTokens[0]);
		//重新构造验证签名
		String expectedTokenSignature = makeTokenSignature(tokenExpiryTime,
				userDetails.getUsername(), userDetails.getPassword());
		//比较重新构造的签名和cookie中数组第三位中的签名是否一致
		if (!equals(expectedTokenSignature, cookieTokens[2])) {
			throw new InvalidCookieException("Cookie token[2] contained signature '"
					+ cookieTokens[2] + "' but expected '" + expectedTokenSignature + "'");
		}

		return userDetails;
	}


```

	上面的方法就是根据cookie中查询的用户名到数据库中查询出真实用户信息。根据真实的信息，构造出签名与cookie中的签名进行比较。
	这样可以防止cookie伪造，但不能防止cookie被别人窃取使用。

### 2. 使用PersistentTokenBasedRememberMeServices如何实现processAutoLoginCookie方法的
```

protected UserDetails processAutoLoginCookie(String[] cookieTokens,
			HttpServletRequest request, HttpServletResponse response) {

		if (cookieTokens.length != 2) {
			throw new InvalidCookieException("Cookie token did not contain " + 2
					+ " tokens, but contained '" + Arrays.asList(cookieTokens) + "'");
		}
		//获取cookie中的series和token。series是标识唯一cookie，token是生成cookie时创建的随机值
		final String presentedSeries = cookieTokens[0];
		final String presentedToken = cookieTokens[1];
		//根据series到数据库中查询出此cookie的信息
		PersistentRememberMeToken token = tokenRepository
				.getTokenForSeries(presentedSeries);

		if (token == null) {
			// No series match, so we can't authenticate using this cookie
			throw new RememberMeAuthenticationException(
					"No persistent token found for series id: " + presentedSeries);
		}

		//比较数据库中查询出的token中的Token值是否与cookie中的token值相同，防止cookie伪造
		if (!presentedToken.equals(token.getTokenValue())) {
			// Token doesn't match series value. Delete all logins for this user and throw
			// an exception to warn them.
			tokenRepository.removeUserTokens(token.getUsername());

			throw new CookieTheftException(
					messages.getMessage(
							"PersistentTokenBasedRememberMeServices.cookieStolen",
							"Invalid remember-me token (Series/token) mismatch. Implies previous cookie theft attack."));
		}
		//验证是否过期
		if (token.getDate().getTime() + getTokenValiditySeconds() * 1000L < System
				.currentTimeMillis()) {
			throw new RememberMeAuthenticationException("Remember-me login has expired");
		}

		//上面两步验证说明，cookie是有效的，这里创建新的token（Series不变，Token变成随机值）
		PersistentRememberMeToken newToken = new PersistentRememberMeToken(
				token.getUsername(), token.getSeries(), generateTokenData(), new Date());

		try {
		//将修改后的Token更新到数据库 和返回给前台
			tokenRepository.updateToken(newToken.getSeries(), newToken.getTokenValue(),
					newToken.getDate());
			addCookie(newToken, request, response);
		}
		catch (Exception e) {
			logger.error("Failed to update token: ", e);
			throw new RememberMeAuthenticationException(
					"Autologin failed due to data access problem");
		}
		return getUserDetailsService().loadUserByUsername(token.getUsername());
	}

```

	这个rememberMeServices的处理逻辑是，每次自动登录成功后将cookie中的某个随机值和数据库同步更新，假设cookie别别人盗用，自动登录后盗用者的cookie被更新了。
	主人的cookie就会变无效。下次主人会自动登录失败，系统就能发现cookie被盗用，此时删除数据库中的对应cookie验证，通知用户改密码等.