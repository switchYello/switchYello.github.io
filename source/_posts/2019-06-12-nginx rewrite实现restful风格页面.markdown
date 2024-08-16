---
layout:     post
title:      "nginx rewrite实现restful风格页面"
subtitle:   ""
date:       2019-06-12
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - nginx
---


# nginx rewrite实现restful风格页面

## 问题描述
	
	问题描述： 现在后端采用nginx 动静分离。某一页面地址如“www.example.com/article.html?page=1”所示，通过get参数传递到页面，再在页面上解析参数，通过ajax获取到内容展示出来。
	但该类网址是url相同的只是参数不同。想要改成如“www.example.com/page1.html” 这种类型的url,页面还是同一个页面，但是看起来像拥有多个页面一样。


## 解决思路
	
	让nginx 解析url，匹配出所有的 `www.example.com/page*.html` 形式的url，统统返回给前端相同的页面。

## nginx 配置

``` nginx
// 这是以前的配置，所有静态页面从指定文件夹寻找
location / {
	root /home/html/;
}
		
```

```
// 修改后的配置如下
location / {
	rewrite /page([0-9]+?)\.html /article.html break;
	root /home/html/;
}
		
```

## 原理
	
	rewrite [正则表达式 替换前的URL] [替换后的URL] [标识];
	在匹配资源时将所有rewrite第一个参数正则匹配的链接，替换成第二个参数指定的地址。
	我这里是将 /page(若干个数字).html 替换成 /article.html ，这样就符合要求了

## 标识的类型和解释


标识可取以下几种类型
	 
|标识|作用|
--|--
last|停止rewrite的检测
break|停止rewrite的检测
redirect|返回302临时重定向，浏览器重新跳转
permanent|返回301永久重定向，浏览器会重新跳转


** break 表示进行rewrite完成后，拿着新的url在该location内部进行匹配，也就是在下面的root里面去寻找资源。**

** last 不但在该location下进行匹配，还会跳出该location 去其他location进行匹配。**
 
 
	正则表达式可以自由发挥，功能很强到多数都是支持的，
	如果访问不通，可以去nginx下面的log/error.log 里面找找日志看看
	

## 参考文件

[Nginx高级之Rewrite规则](https://blog.csdn.net/ip_JL/article/details/84422124)

[正则表达式](https://www.runoob.com/regexp/regexp-syntax.html)