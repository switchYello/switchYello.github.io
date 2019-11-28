---
layout:     post
title:      "springboot 添加defaultServlet实现更好的文件下载功能"
subtitle:   ""
date:       2019-10-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springboot
---


### springboot 添加defaultServlet实现更好的文件下载功能

	defaultServlet是tomcat为我们提供的能下载或打开tomcat项目文件夹下的静态文件的servlet，他的url-pattern是 ‘/’
	但如果使用springboot，会自动注册dispatchServlet到容器中 url-pattern也是 ‘/’会把defaultServlet给覆盖掉

#### 下面我们重新注册该servlet到springboot中来处理文件下载

###要求
	实现文件下载和文件打开功能
	文件存储目录在 “d:/files/”文件夹下
	如果访问 “localhost/download/xxx.jpg”则到该‘d:/files/’下查找xxx.jpg下载
	如果访问“localhost/download/aa/bb/cc/xxx.jpg” 则到“d:/files/aa/bb/cc/”下查找
	如果访问“localhost/static/xxx.jpg” 则在“d:/files/xxx.jpg”在浏览器打开而不是下载


### 过程

```
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