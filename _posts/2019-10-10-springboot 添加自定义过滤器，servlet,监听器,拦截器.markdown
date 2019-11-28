---
layout:     post
title:      "springboot 添加自定义过滤器，servlet,监听器,拦截器"
subtitle:   ""
date:       2019-10-10
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springboot
---


## springboot 添加自定义过滤器，servlet,监听器,拦截器

### 开发阶段需要添加一个过滤器来做一些log，方便调试，如打印方法执行时间，方法参数等信息

## 1.添加过滤器

```
@Configuration
public class FilterConfig {

	//配置filter，默认filter的顺序是最后执行的，可以修改，这里不进行修改
	//springboot 默认会添加一些编码相关的filter在最前面，如果该顺序不要改太小，OrderedCharacterEncodingFilter的顺序是 `Ordered.HIGHEST_PRECEDENCE`
	//这里使用默认顺序
	//注意UrlPatterns不是ant路径模式，不要写成了 `/**`否则不生效
    @Bean
    public FilterRegistrationBean filterRegistrationBean() {
        FilterRegistrationBean<Filter> register = new FilterRegistrationBean<>();
        register.setFilter(new TimeFilter());
        register.addUrlPatterns("/*");
        return register;
    }


	//过滤器器打印请求路径和参数，方法执行完成统计执行毫秒值
    private static class TimeFilter implements Filter {
        private static Logger log = LoggerFactory.getLogger(TimeFilter.class);

        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
            HttpServletRequest req = (HttpServletRequest) request;
            long start = System.currentTimeMillis();
            log.info("{} {}", req.getServletPath(), JSONObject.toJSONString(request.getParameterMap()));
            chain.doFilter(request, response);
            log.info("{} {}ms", req.getServletPath(), System.currentTimeMillis() - start);
        }
    }
}

```

## ！！同理 相同的办法可以添加自定义servlet listener等，分别构造出一个 `ServletRegistrationBean` , `FilterRegistrationBean`,`ServletListenerRegistrationBean`
## 



## 2.添加拦截器

```
//继承WebMvcConfigurationSupport
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {

    //重写此方法
    //注意此处的路径格式是ant模式
       @Override
    protected void addInterceptors(InterceptorRegistry reg) {
        reg.addInterceptor(new loginInterceptor())
                .addPathPatterns("/**");
    }


    private static class loginInterceptor implements HandlerInterceptor {
        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            System.out.println("请求经过拦截器");
            return true;
        }
    }


}

```