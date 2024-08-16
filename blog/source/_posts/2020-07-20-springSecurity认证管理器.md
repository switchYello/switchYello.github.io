---
layout:     post
title:      "springSecurity认证管理器"
subtitle:   ""
date:       2020-07-20
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- springSecurity
- AuthenticationManager
typora-root-url: ..
---



# springSecurity认证管理器

​	

​		SpringSecurity负责验证用户身份需要用到认证管理器（AuthenticationManager）。

认证管理器的接口很简单，只有一个方法，将用户名密码传入此认证器，如果不报错则为认证通过。

```java
public interface AuthenticationManager {
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;
}
```



## 认证管理器

​		在springSecurity里，我们可以配置多个认证管理器，将多个认证管理器合并成一个`ProviderManager`对外提供服务，只要有一个认证管理器认证通过，则表示通过，是一种责任链模式。



​	![image-20200720125330662](/img/in/2020-07-20-springSecurity认证管理器/image-20200720125330662.png)





​		将多个authenticationProviders合并成一个的代码实现在这里

```java
//org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder#performBuild 	
@Override
	protected ProviderManager performBuild() throws Exception {
        //将多个authenticationProviders合并成一个
		ProviderManager providerManager = new ProviderManager(authenticationProviders,
				parentAuthenticationManager);
        //验证通过是否擦除密码
		if (eraseCredentials != null) {
			providerManager.setEraseCredentialsAfterAuthentication(eraseCredentials);
		}
		if (eventPublisher != null) {
			providerManager.setAuthenticationEventPublisher(eventPublisher);
		}
		providerManager = postProcess(providerManager);
		return providerManager;
	}
```



## 配置认证管理器

配置的方式有两种，一种是重写下面的方法，配置好`userDetailService` 和 `passwoedEncoder`。

```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(new UserDetailsService() {
            @Override
            public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
                //此处从数据库加载用户信息，并构造出userDetail的实例返回
                return null;
            }
        }).passwordEncoder(new PasswordEncoder() {
            @Override
            public String encode(CharSequence rawPassword) {
                return null;
            }

            //验证前传入的密码和数据库里的密码是否正确
            @Override
            public boolean matches(CharSequence rawPassword, String encodedPassword) {
                return false;
            }
        });

    }
```

上面的代码则会自动配置一个`DaoAuthentication`。





​	另一种方式是将`userDetailService` 和 `passwordEncoder` 配置成spring的bean，框架就能自动查找到。

具体实现请查看在`@EnableGlobalAuthentication`注解是如何做的。

```java
//配置初始化userDetail的配置
@Bean
public static InitializeUserDetailsBeanManagerConfigurer initializeUserDetailsBeanManagerConfigurer(ApplicationContext context) {
		return new InitializeUserDetailsBeanManagerConfigurer(context);
	}

//配置会被自动注入进来
@Autowired(required = false)
public void setGlobalAuthenticationConfigurers(
    List<GlobalAuthenticationConfigurerAdapter> configurers) {
    configurers.sort(AnnotationAwareOrderComparator.INSTANCE);
    this.globalAuthConfigurers = configurers;
}

//配置内部的实现
class InitializeUserDetailsManagerConfigurer
			extends GlobalAuthenticationConfigurerAdapter {
		@Override
		public void configure(AuthenticationManagerBuilder auth) throws Exception {
			if (auth.isConfigured()) {
				return;
			}
            //从容器内获取UserDetailsService
			UserDetailsService userDetailsService = getBeanOrNull(
					UserDetailsService.class);
			if (userDetailsService == null) {
				return;
			}
			//从容器内获取PasswordEncoder
			PasswordEncoder passwordEncoder = getBeanOrNull(PasswordEncoder.class);
			UserDetailsPasswordService passwordManager = getBeanOrNull(UserDetailsPasswordService.class);
		
            //创建DaoAuthenticationProvider
			DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
			provider.setUserDetailsService(userDetailsService);
			if (passwordEncoder != null) {
				provider.setPasswordEncoder(passwordEncoder);
			}
			if (passwordManager != null) {
				provider.setUserDetailsPasswordService(passwordManager);
			}
			provider.afterPropertiesSet();

			auth.authenticationProvider(provider);
		}


```



​	上面两种配置方式都是可以的，本人更推荐第一种方式。一方面代码几种在一个配置文件里，比较方便查找，一方面如果想配置多个`HttpSecurity`，每个有独立的`UserDetailService`也更方便。





## 框架对两种配置方式如何选择的

​	查看代码

```java
org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter#authenticationManager
```

​	通过`configure`方法重写，如果没重写则为自动配置，重写了就是手动配置。

```java
	protected AuthenticationManager authenticationManager() throws Exception {
		//如果没初始化则初始化
        if (!authenticationManagerInitialized) {
		//默认将disableLocalConfigureAuthenticationBldr设置true
            configure(localConfigureAuthenticationBldr);
		//使用本地自定义配置，就是配置1
			if (disableLocalConfigureAuthenticationBldr) {
				authenticationManager = authenticationConfiguration
						.getAuthenticationManager();
			}
		//使用自动配置，配置2
			else {
				authenticationManager = localConfigureAuthenticationBldr.build();
			}
			authenticationManagerInitialized = true;
		}
		return authenticationManager;
	}
	
		protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		this.disableLocalConfigureAuthenticationBldr = true;
	}
```







