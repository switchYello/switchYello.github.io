---
layout:     post
title:      "servlet-mapping 路径配置 斜杠、斜杠星区别"
subtitle:   ""
date:       2019-07-23
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - java
---


# servlet-mapping 路径配置怎么配， `/` `/*` 区别

## 1.匹配规则

	/xx 表示精准匹配xx路径

	*.action 表示匹配action为后缀的
	
	/ 表示匹配所有路径，优先级较低
	
	/* 匹配所有路径 优先级小于 /xx这种精准匹配，小于/xx/*这种路径比他更为精准的匹配 大于*.action这种后缀匹配，这将导致一些问题

---------------------------
## 2.默认servlet处理静态资源和jsp，为什么什么都不写也能访问静态资源和jsp
	
	因为: tomcat容器自带两个servlet，请查看tomcat的web.xml（非项目的），里面写着两个servlet分别是


  
	//默认的servlet 名字叫default
	<servlet>
	<servlet-name>default</servlet-name>
	<servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
	<init-param>
	  <param-name>debug</param-name>
	  <param-value>0</param-value>
	</init-param>
	<init-param>
	  <param-name>listings</param-name>
	  <param-value>false</param-value>
	</init-param>
	<load-on-startup>1</load-on-startup>
	</servlet>
	
	//默认的servlet 名字叫jsp
	<servlet>
	<servlet-name>jsp</servlet-name>
	<servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
	<init-param>
	  <param-name>classdebuginfo</param-name>
	  <param-value>true</param-value>
	</init-param>
	<init-param>
	  <param-name>fork</param-name>
	  <param-value>false</param-value>
	</init-param>
	<init-param>
	  <param-name>xpoweredBy</param-name>
	  <param-value>false</param-value>
	</init-param>
	<load-on-startup>3</load-on-startup>
	</servlet>



## 他们拦截路径分别是

	<servlet-mapping>
	<servlet-name>default</servlet-name>
	<url-pattern>/</url-pattern>
	</servlet-mapping>
	//jsp 拦截
	<servlet-mapping>
	<servlet-name>jsp</servlet-name>
	<url-pattern>*.jsp</url-pattern>
	<url-pattern>*.jspx</url-pattern>
	</servlet-mapping>




> 可以看出default servlet 默认情况下处理webapp目录下的静态文件，比如我们放一张图片或一个css文件在webapp目录下可以直接访问
  jsp servlet 处理webapp目录下的*.jsp文件，这样我们不用写任何代码就能访问jsp文件
	

---------------------------------

## 3.配置springmvc，如何不影响静态资源访问
	

### 3.1 配置方式1

	<servlet-mapping>
		<servlet-name>dispatcher</servlet-name>
		<url-pattern>*.do</url-pattern>
	</servlet-mapping>	


	我们配置springmvc为  *.do 等行使的路径。这样可以将我们的请求与tomcat默认的分开，相互不影响。
	
### 3.2 配置方式2 ，我们不希望带任何后缀

	<servlet-mapping>
		<servlet-name>dispatcher</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>	

	相当于覆盖了tomcat的default servlet，这样不需要任何后缀就可以访问dispatcher，但是此时带来的后果就是，无法处理静态资源.
	此时放在webapp目录下的css js jpg 等文件没有servlet处理，所以无法访问

#### 3.2.1解决办法
	
	我们可以将静态资源后缀指定使用default servlet拦截
	
	<servlet-mapping>
		<servlet-name>default</servlet-name>
		<url-pattern>*.jpg</url-pattern>
	</servlet-mapping>	
	<servlet-mapping>
		<servlet-name>default</servlet-name>
		<url-pattern>*.css</url-pattern>
	</servlet-mapping>

	这样将常用后缀的静态资源主动交给deafault处理，就可以了


## 4 /*带来的问题

	一般不要将拦截范围设为 /*他将导致拦截范围过于宽，比如如下这样设置

	<servlet-mapping>
		<servlet-name>A</servlet-name>
		<url-pattern>/test</url-pattern>
	</servlet-mapping>
	
	<servlet-mapping>
		<servlet-name>A1</servlet-name>
		<url-pattern>/test/*</url-pattern>
	</servlet-mapping>
	
	<servlet-mapping>
		<servlet-name>B</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
	
	<servlet-mapping>
		<servlet-name>C</servlet-name>
		<url-pattern>*.do</url-pattern>
	</servlet-mapping>
	
	<servlet-mapping>
		<servlet-name>D</servlet-name>
		<url-pattern>/*</url-pattern>
	</servlet-mapping>

	
	
#### 得到结果 请求

	localhost/test 		-->A
	localhost/test/xyz 	-->A1
	localhost/xyz   	-->D
	localhost/xyz.do    -->D
	localhost/   		-->D
	

	可以看到 /*的拦截优先级太高，
	除了精准匹配和次精准匹配外，其他如 *.do 这种类型不起作用，/也不起作用。
	从而导致默认静态文件无法访问，默认jsp文件无法访问，servlet请求转发到jsp无法实现。
	从而各种404

	所以千万不要使用 /*， 不想要接口链接有后缀的，可以用 3.2节的解决办法即可