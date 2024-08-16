---
layout:     post
title:      "springmvc put请求参数绑定"
subtitle:   ""
date:       2019-11-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springboot
---


### springmvc put请求参数绑定

#### 现象

	使用put请求 + x-www-urlencoded的形式，后端无法获取到参数


* 对于post请求，tomcat会帮忙解析请体中的参数封装成request，传入servlet，支持`form-data` 和 `x-www-urlencoded`两种形式的请求

* 对于put请求，tomcat只会帮忙解析form-data类型的请求，x-www-urlencoded形式的会以request.getInputStream的形获取



	要想让springmvc支持put + x-www-urlencoded形式的请求，
	1.要么配置tomcat让tomcat帮忙解析，
	2.要么添加`HttpPutFormContentFilter`拦截器（新版的叫做`FormContentFilter`支持delete请求），
	该拦截器做的事情是从request.getInputStream读取数据，解析成键值对封装到request中。


#### 对于application/json形式的请求

	这种请求tomcat也是不帮助解析的，但springMvc自带了很多messageCovery解析这种形式的参数。
	
	原理是框架碰到了带有@RequestBody注解的参数，会尝试从request.getInputStream中读取数据
	然后使用自带的messageCovery进行解析从流中读取到的内容为参数

	
	通常我们会配置JackSon的messageCovery来处理json数据，反序列化成对象。