---

layout:     post
title:      "SpringBoot集成Swagger实战"
subtitle:   ""
date:       2020-04-03
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- java
- springboot
- swagger
typora-root-url: ..
---



* TOC
{:toc}


## 为什么要使用Swagger 

​	作为后端，写完接口有时会忘记维护`Api`文档，时间长了接口文档和代码就相差甚远，导致文档难以维护且失去意义。使用`Swagger`，可以在写接口时顺便维护文档，从而接口变文档也变。并且文档页面还可以在线测试，且比手动写`Markdown`更美观。

​	这篇文章将讲解如何在`SpringBoot`项目里面使用`Swagger`。



## 1.创建`Springboot`项目



### 	创建空的`SpringBoot`项目

​		使用`idea`初始化`SpringBoot`项目，因为是web项目，别忘了勾选`Spring Web`依赖。



![image-20200403132147933](/img/in/2020-04-03-springboot集成swagger实战/image-20200403132147933.png)



### 	添加`controller`

​	创建一个如下增删改查的`Controller`

```java
@RestController
@RequestMapping("test")
public class TestController {

    @PostMapping("create")
    public int create(String name, Integer age) {
        return 1;
    }

    @PostMapping("delete")
    public int delete(int id) {
        return 1;
    }

    @PostMapping("update")
    public int update(int id, String name, Integer age) {
        return 1;
    }

    @GetMapping("get_all")
    public List<String> get() {
        return Arrays.asList("张三", "李四", "赵六");
    }
}
```



### 	测试接口

​		在浏览器里顺利打开接口

![image-20200403132815065](/img/in/2020-04-03-springboot集成swagger实战/image-20200403132815065.png)



## 2.添加`Swagger`

### 添加依赖

```xml
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>
```



### 编写`Swagger`配置

​	`Springfox`需要编写一个`Docket`的`Bean`，在这个类里面做配置，下面是新建一个配置类，让`SpringBoot`扫描到这个`Bean`并注入到容器。



```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {

    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any())
                .build();

    }
 }
```



`@EnableSwagger2` 就是启动`Swagger`功能。



### 访问验证

​	浏览器访问`http://localhost:8080/v2/api-docs`,可以看到输出的一系列 `Json`，可读性太差，但是`Swagger`提供了一个可视化界面，方便我们阅读。



## 3.添加`Swagger-Ui`

### 添加依赖

```xml
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
```



### 访问

​	访问`http://localhost:8080/swagger-ui.html`我们写的四个接口都在这里

![image-20200403134554617](/img/in/2020-04-03-springboot集成swagger实战/image-20200403134554617.png)



### 配置

#### 		配置文档标题和版本

​		配置`Swagger`需要提供一个`ApiInfo`类描述文档标题，版本，用户名，邮箱等信息,像下面这样添加到`Docket`里面。

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {

    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any())
                .build().apiInfo(apiInfo());

    }

    private ApiInfo apiInfo() {
        return new ApiInfo("测试开发文档", "开发文档描述", "0.01",
                null, null,
                "", "", Collections.emptyList());
    }

}
```



​	界面变成了这样，文档名、版本、描述都变了。

![image-20200403134110601](/img/in/2020-04-03-springboot集成swagger实战/image-20200403134110601.png)



#### 	配置Controller描述

​		类上添加注解`@Api`，并填写`tags`可以配置`controller`描述

```java
    @RestController
    @RequestMapping("test")
    @Api(tags = "测试相关接口")
    public class TestController {
```



#### 	配置接口名

​		给方法添加注解，可以配置接口名称，添加` @ApiOperation`注解在接口方法上。

```java
    @ApiOperation("获取所有数据")
    @GetMapping("get_all")
    public List<String> get() {
        return Arrays.asList("张三", "李四", "赵六");
    }
```



#### 	配置参数名

​		使用注解`@ApiImplicitParams`可以配置多个参数，使用注解`@ApiImplicitParam`可以配置一个参数。

```java
    @ApiOperation("根据id更新")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id", value = "用户id", required = true),
            @ApiImplicitParam(name = "name", value = "用户名"),
            @ApiImplicitParam(name = "age", value = "用户年龄")
    })
    @PostMapping("update")
    public int update(int id, String name, Integer age) {
        return 1;
    }
```

​	`required`表示是否必填，默认是`false`。



#### 	验证

​		页面变成了这样，`controller`变成了`@Api`标记的名称，每个接口名变成了`@ApiOperation`指定的名称，

参数描述变成了`@ApiImplicitParam`指定的`value`。



![image-20200403135331799](/img/in/2020-04-03-springboot集成swagger实战/image-20200403135331799.png)





### 更多配置

#### 	`@ApiParam` 注解



​		除了使用`@ApiImplicitParam`外还可以使用`@ApiParam`标记参数。这样可以省去`name`属性，但是可能会让方法参数列表看起来混乱。

```java
    @ApiOperation("根据id删除")
    @PostMapping("delete")
    public int delete(@ApiParam(value = "要删除的用户id", required = true) int id) {
        return 1;
    }
```



​		当然这两种方式可以混合使用，只要不重复标记到同一个参数上就行。



## 	4.在线测试接口

​		找到一个接口，点击Try it out。

![image-20200403140401482](/img/in/2020-04-03-springboot集成swagger实战/image-20200403140401482.png)



​	页面就会变成可编辑形式，点击Execute，就可以按照配置的参数发送请求了，并且下面会显示执行结果。

![image-20200403140446596](/img/in/2020-04-03-springboot集成swagger实战/image-20200403140446596.png)



## 5.注意事项

### 文件上传写法



​		有是后端需要处理上传文件，则请使用`@ApiParam`注解，不要用`@ApiImplicitParam`,这样在页面上测试时就能直接鼠标选择要上传的文件了。

​		这两个注解作用于`MultipartFile`类型上的区别，读者可以亲自尝试对比下。

```java

    @RequestMapping("upload")
    @ApiOperation("上传文件")
    public int uploadFile(@ApiParam("需要上传的文件") MultipartFile file) {
        return 1;
    }
```





### 需要指定请求方式

​		像下面这两种指定请求Method方式都可以。


```java
	@GetMapping("upload")
	@RequestMapping(value = "upload",method = RequestMethod.GET)
```



​		如果不指定请求方式，`Swagger`将尝试使用所有请求方式去调用接口，文档页面看起来会很乱的。

下面`upload`接口就没指定请求方式，`Swagger`显示是这样的



![image-20200403140957368](/img/in/2020-04-03-springboot集成swagger实战/image-20200403140957368.png)



## 6.结束语

​	本文就介绍到这里，基本能满足我们的需求，我们在修改接口时，顺便修改下注解就可以。同时前端在使用接口文档时，也可以在页面上直接测试接口。