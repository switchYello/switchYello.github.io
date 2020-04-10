---

layout:     post
title:      "SpringBoot访问静态文件的几种方法"
subtitle:   ""
date:       2020-04-10
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- java
- springboot
- defaultServlet
typora-root-url: ..
---



* TOC
{:toc}
​        我以前写过一个使用`DefaultServlet`实现下载的博客[springboot 添加defaultServlet实现更好的文件下载功能](https://www.huangchaoyu.com/2019/10/15/springboot-%E6%B7%BB%E5%8A%A0defaultServlet%E5%AE%9E%E7%8E%B0%E6%9B%B4%E5%A5%BD%E7%9A%84%E6%96%87%E4%BB%B6%E4%B8%8B%E8%BD%BD%E5%8A%9F%E8%83%BD/) ,这篇文章是对这类问题的多种解法讲解。



# 目的

​         将图片存储在`f:\\test`文件夹或其子文件夹下，实现访问`http://localhost/static/1.jpg`网页上展示图片，访问`http://localhost/download/1.jpg`浏览器下载图片。



# 方法1

​        请参考这篇文章的实现， [springboot 添加defaultServlet实现更好的文件下载功能](https://www.huangchaoyu.com/2019/10/15/springboot-%E6%B7%BB%E5%8A%A0defaultServlet%E5%AE%9E%E7%8E%B0%E6%9B%B4%E5%A5%BD%E7%9A%84%E6%96%87%E4%BB%B6%E4%B8%8B%E8%BD%BD%E5%8A%9F%E8%83%BD/)，原理是配置一个`DefaultServlet`来处理所有静态资源请求。

​       

# 方法2

​    原理使用`Controller`拦截掉静态资源的请求，转发到`DefaultServlet`。

​    配置`Controller如下`，使用`RequestMapping`分开配置，处理下载的`/download/**` 和处理展示的 `/static/**` 分开。

​    他们俩的唯一区别是，`/download/**` 手动设置了响应头，在此设置的`content-type`,后面就不会被设置了，否则会根据资源类型自适应`content-type`。

​    调用`ServletContext`的`Dispatcher`做转发。

```java
@RestController
public class ResourceController implements ServletContextAware {
	
	//拦截资源，设置响应头application/octet-stream，浏览器表现为下载
    @RequestMapping("/download/**")
    public void download(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.setContentType("application/octet-stream");
        RequestDispatcher rd = req.getServletContext().getNamedDispatcher("default");
        rd.forward(req, resp);
    }
	
	//拦截static/**，不覆盖响应头，根据类型自适应响应头
    @RequestMapping("/static/**")
    public void staticSource(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        RequestDispatcher rd = req.getServletContext().getNamedDispatcher("default");
        rd.forward(req, resp);
    }

    //继承ServletContextAware，此方法内配置
    //文件根目录，需要删除的虚拟路径
    @Override
    public void setServletContext(ServletContext servletContext) {
        WebResourceRoot attribute = (WebResourceRoot) servletContext.getAttribute(Globals.RESOURCES_ATTR);
        String basePath = "F:\\test";

        
        //假设请求连接是 http://localhost/static/abc/123.jpg
        //则实际查找路径是 F:\\test/abc/123.jpg
        //相当于拿path 减去 下面的第二个参数，再拼接在basePath后面
        attribute.addPreResources(new DirResourceSet(attribute, "/static", basePath, "/"));
        attribute.addPreResources(new DirResourceSet(attribute, "/download", basePath, "/"));
    }
}
```



​        上面的做法是在`Controller`里面拦截请求，再请求转发到`DefaultServlet`上，这样就不用配置`DefaultServlet`的`ServletMapper`了，使用`SpringMvc`来管理路径问题。

​        这种方法默认`defaultServlet`被配置（`Tomcat`环境下是会被配置的）。



# 方法3

​        本方法使用`Spring`的实现，即使底层用的不是`Tomcat`，`Spring`也能将这种差异屏蔽。

​        `SpringBoot`默认会按顺序扫描以下目录的静态资源文件，这是`SpringBoot`默认配置的，默认拦截路径是`/**`。

```java
"classpath:/META-INF/resources/",
"classpath:/resources/",
"classpath:/static/",
"classpath:/public/",
"/"
```



​        写一个类继承`WebMvcConfigurationSupport`，实现`addResourceHandlers`方法

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {

    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        //文件写法
        registry.addResourceHandler("/static/**", "/download/**")
                .addResourceLocations("file:F:/test/");
        //classpath 写法
        registry.addResourceHandler("/classpath/**")
                .addResourceLocations("classpath:/static/", "classpath:/META-INF/resources/");

    }
}
```

​     

​        **要注意的点：**

- 资源在外部文件夹下的路径写法要以`file:`开头，以`/`结尾，如`file:D:/test/`

- 访问`classpath`的资源，路径要以`classpath:/`开头，以`/`结尾,如`classpath:/static/`

- 访问`第三方jar`内部的静态文件，配置是`classpath:/META-INF/resources/`

- 像下面这样配置两个相同的路径，则后面覆盖前面的。

  - ```java
    registry.addResourceHandler("/classpath/**").addResourceLocations("file:D:/test")
                                                                     registry.addResourceHandler("/classpath/**").addResourceLocations("file:E:/zz")
    ```

- **切记不要配置成`classpath:/`，这会导致`resources`文件夹暴露出来，导致配置文件泄露**



​        但是这种方法并不能实现文件下载，能识别出`content-type`的如图片会自动打开在浏览器上，不能识别的资源才会调用浏览器下载。



# 关于自动配置

- 自动配置

  ​       一旦继承了`WebMvcConfigurationSupport`类，即使不重写`addResourceHandlers`类，自动配置的5个路径就失效了。所以就不能访问静态文件了。

​       

- `swagger-ui`的静态资源

  ​     在使用`swagger-ui`时，它的原理是将`swagger-ui.html`页面打包到`springfox.swagger-ui`里面了。

  要想访问这个页面需要配置下面的路径来处理。

  ```java
  registry.addResourceHandler("/**").addResourceLocations("classpath:/META-INF/resources/");
  ```

  

- 方法3搭配方法2

  举个例子:

  ​		1. 将`/static`路径进行映射。

  ```java
   registry.addResourceHandler("/static/**").addResourceLocations("classpath:/static/");
  ```

  

  ​      2. 将参数中的文件名重新`forward`，这样也能实现根据参数提供静态文件名。

  ```java
     @GetMapping("static")
      public void test(HttpServletRequest res, HttpServletResponse resp, String fileName) throws ServletException, IOException {
          res.getServletContext().getRequestDispatcher("/classpath/"+fileName).forward(res, resp);
      }
  ```

  

  