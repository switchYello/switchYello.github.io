---
layout:     post
title:      "2020-12-04-springboot注册拦截器不生效"
subtitle:   ""
date:       2020-12-04
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- bug
typora-root-url: ..
---



# springboot注册拦截器不生效



​		通常注册filter是向下面这样注册的，但是如果使用lambda又没有写filter名字就会导致后面的不生效，因为注册时是按照名字作为key存入map里面的，如果已经注册后面同名的就会忽略。

如果没有给名字，名字就是filter的类名，而lambda的类型是object类，所以是同名的。



```java
    @Bean
    public FilterRegistrationBean<Filter> timeFilter() {
        FilterRegistrationBean<Filter> reg = new FilterRegistrationBean<>();
        reg.setFilter((request, response, chain) -> {
            HttpServletRequest req = (HttpServletRequest) request;
            try {
                chain.doFilter(request, response);
            } finally {
                log.info("1111");
            }
        });
        reg.addUrlPatterns("/*");
        return reg;
    }
	
	@Bean
    public FilterRegistrationBean<Filter> timeFilter() {
        FilterRegistrationBean<Filter> reg = new FilterRegistrationBean<>();
        reg.setFilter((request, response, chain) -> {
            HttpServletRequest req = (HttpServletRequest) request;
            try {
                chain.doFilter(request, response);
            } finally {
                log.info("222");
            }
        });
        reg.addUrlPatterns("/*");
        return reg;
    }
```



上面这样写，第二个是不生效的，正确的做法是向下面这样，给一个名字

```java
	@Bean
    public FilterRegistrationBean<Filter> timeFilter() {
        FilterRegistrationBean<Filter> reg = new FilterRegistrationBean<>();
        reg.setFilter((request, response, chain) -> {
            HttpServletRequest req = (HttpServletRequest) request;
            try {
                chain.doFilter(request, response);
            } finally {
                log.info("1111");
            }
        });
        reg.addUrlPatterns("/*");
        reg.setName("filter1");
        return reg;
    }
```










