---
layout:     post
title:      "SpringSecurityConf配置"
subtitle:   ""
date:       2019-10-29
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springSecurity
---


## SpringSecurityConf配置


### 上配置文件
```java
//开启配置
@EnableWebSecurity
public class SpringSecurityConf extends WebSecurityConfigurerAdapter {
	
	//自己写的userService
    @Resource
    private UserService userService;

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        //解决跨域问题，在此处配置就不用在WebMvcConfigurationSupport里面配置了
        http.cors().configurationSource(request -> {
            CorsConfiguration c = new CorsConfiguration();
            c.setAllowedOrigins(Arrays.asList("http://192.168.31.114", "*"));
            c.setAllowedMethods(Arrays.asList("POST", "GET", "OPTIONS", "PUT"));
            c.setAllowCredentials(true);
            c.setMaxAge(1800L);
            return c;
        });

        //禁用headerFilter 的禁止iframe访问策略
        http.headers().frameOptions().disable();
		http.headers().cacheControl().disable();
        //禁止csrf
        http.csrf().disable();
        //退出登录逻辑
        http.logout()
                .logoutUrl("/system/logout")
                .logoutSuccessHandler((request, response, authentication) -> {
                    String str = "{\"s\":\"1\",\"r\":\"login out success\"}";
                    sendOut(str, response);
                });
        //登录post，登录成功，登录失败 处理逻辑
        http.formLogin().loginPage("/").loginProcessingUrl("/system/login")
                .successHandler((request, response, authentication) -> {
                    String str = "{\"s\":\"1\",\"r\":\"login success\"}";
                    sendOut(str, response);
                })
                .failureHandler((request, response, exception) -> {
                    String str = "{\"s\":\"0\",\"r\":\"" + exception.getMessage() + "\"}";
                    sendOut(str, response);
                });
        //未登陆用户访问接口回调逻辑
        http.exceptionHandling().authenticationEntryPoint((request, response, authException) -> sendOut("{\"s\":\"0\",\"r\":\"not login\"}", response));
        //记住我功能，两个key要设置成一样的
        TokenBasedRememberMeServices rememberMeServices = new TokenBasedRememberMeServices("TokenBasedRememberMeServicesKey$#&^$", userDetailsService);
        rememberMeServices.setAlwaysRemember(true);
        rememberMeServices.setCookieName("system.rememberMe");
        http.rememberMe().rememberMeServices(rememberMeServices).key("TokenBasedRememberMeServicesKey$#&^$");

        //权限路径
        http.authorizeRequests().antMatchers("/", "/login", "/**/*.ico").permitAll();
        http.authorizeRequests().anyRequest().authenticated();
    }

    //这个类配置认证管理器的AuthenticationManagerBuilder
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder);
    }


    //框架会将此service 包装成一个DaoAuthenticationProvider
    private UserDetailsService userDetailsService = username -> {
		//从数据库查
        com.test.User userByUserName = userService.getUserByUserName(username);
        //只允许登录管理员类型的账户
        if (userByUserName == null) {
            throw new UsernameNotFoundException(username);
        }
		//UserAdv实现了UserDetails
        return new UserAdv(userByUserName);
    };
	
    private PasswordEncoder passwordEncoder = new PasswordEncoder() {
		//明文转密文的方法，此处自己写的基于md5的转换方式        
		@Override
        public String encode(CharSequence rawPassword) {
            return UserService.securityPassword(rawPassword.toString());
        }

        /**
         *
         * @param rawPassword 用户上传的密码
         * @param encodedPassword userDetailsService查询的密码
         * @return 决定是否匹配, 这里只要上传的密码进行加密后的结果等于查询出的密码，则说明匹配
         */
        @Override
        public boolean matches(CharSequence rawPassword, String encodedPassword) {
            return encode(rawPassword).equals(encodedPassword);
        }
    };

	//发送json给response
    private static void sendOut(String str, HttpServletResponse response) throws IOException {
        if (response != null && !response.isCommitted()) {
            response.setContentType("application/json;charset=utf-8");
            try (OutputStream out = response.getOutputStream()) {
                out.write(str.getBytes(StandardCharsets.UTF_8));
                response.flushBuffer();
            }
        }
    }

}

```