---
layout:     post
title:      "springboot 添加defaultServlet实现更好的文件下载功能"
subtitle:   ""
date:       2019-10-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springboot
    - defaultServlet
---



# Springboot 添加DefaultServlet实现更好的文件下载功能



​        `defaultServlet`是`tomcat`为我们提供的能下载或打开`tomcat`项目文件夹下的静态文件的`servlet`，他的`url-pattern`是 `/`。
​        但如果使用`springboot`，会自动注册`dispatchServlet`到容器中`url-pattern`也是 `/`会把`defaultServlet`给覆盖掉。



## 实现目的

- 实现文件下载和文件打开功能
- 文件存储目录在 `D:/files/`文件夹下
- 如果访问 `localhost/download/xxx.jpg`则到该`d:/files/`下查找`xxx.jpg`下载
- 如果访问`localhost/download/aa/bb/cc/xxx.jpg` 则到`d:/files/aa/bb/cc/`下查找
- 如果访问`localhost/static/xxx.jpg` 则在`d:/files/xxx.jpg`在浏览器`打开而不是下载`



## 过程

​        我们继承一个`DefaultServlet`并在`init`方法里面将文件存储的目录配置进去。然后将这个`Servlet`添加到`Tomcat`里。

```java
@Configuration
public class ServletConfig {
		/**
		*配置DownloadServlet，拦截/static/* 和 /download/*的请求，且程序启动后就初始化而不是等第一次访问才初始化
		*
		*/
	@Bean
    public ServletRegistrationBean servletRegistrationBean() {
        ServletRegistrationBean<Servlet> register = new ServletRegistrationBean<>();
        DefaultServlet defaultServlet = new DownloadServlet();
        register.setServlet(defaultServlet);
        register.addUrlMappings("/static/*", "/download/*");
        register.setLoadOnStartup(1);
        return register;
    }

	//继承defaultServlet
    private static class DownloadServlet extends DefaultServlet {

        private static final long serialVersionUID = 1537326382082402617L;

        /*
         *  String amount = "/download";  //从文件path中移除此路径,前置路径 如访问路径是 localhost/download/123.jpg,则真实查找路径是'/123.jpg'（即移除'/download')
         *  String basePath = "d:/files"; //文件存储所在的文件夹绝对路径,末尾不要以‘/’结尾
         *  String internalPath = "/"; // 不知道有什么用，DirResourceSet应该没用到
         *  DefaultServlet使用若干个ResourceSet进行读取资源，默认读取tomcat项目根目录下的文件，我们自己在设置两个，分别处理/static的和/download的请求
         * */
        @Override
        public void init() throws ServletException {
            WebResourceRoot attribute = (WebResourceRoot) getServletContext().getAttribute(Globals.RESOURCES_ATTR);

            attribute.addPreResources(new DirResourceSet(attribute, "/static", "d:/files", "/"));
            attribute.addPreResources(new DirResourceSet(attribute, "/download", "d:/files", "/"));
            super.init();
        }

		//如果是/download开头的请求，则设置ContentType为流，否则servlet会自行推理，如图片就会直接打开而不是下载
        @Override
        public void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            if (req.getRequestURI().startsWith("/download")) {
                resp.setContentType("application/octet-stream");
            }
            super.service(req, resp);
        }
    }

}

```

​    这里重写service方法的目的是 如果发现`/download`开头的请求，要设置响应头为`application/octet-stream`，这样浏览器就能下载该文件，而不是展示该文件。



## 结尾

​        以上就是在`SpringMvc`环境下，处理静态资源展示和下载的实现方式。