---
layout:     post
title:      "HeaderWriterFilter内部做了什么，导致静态资源不能访问了"
subtitle:   ""
date:       2019-10-29
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springSecurity
---


## HeaderWriterFilter内部做了什么，导致静态资源不能访问了

	此filter会向相应中添加一些响应头，如果不搞清楚可能出现各种问题，策略如下面


#### ContentTypeOptionsConfig，默认启用
	配置x-content-type-options响应头

	服务器设置该响应头，X-Content-Type-Options: nosniff，则浏览器拒绝MIME不正确的响应，放置MIME攻击


#### XXssConfig  默认 1;mode=block
	X-XSS-Protection 响应头，有三种值
	0： 表示关闭浏览器的XSS防护机制
	1：删除检测到的恶意代码， 如果响应报文中没有看到X-XSS-Protection 字段，那么浏览器就认为X-XSS-Protection配置为1，这是浏览器的默认设置
	1; mode=block：如果检测到恶意代码，在不渲染恶意代码

#### CacheControlConfig  默认启用
	CacheControl响应头，控制缓存

#### HstsConfig  
	Strict-Transport-Security响应头，只有https请求时添加此响应头

	告诉浏览器使用https方式访问

#### FrameOptionsConfig 默认启用
	X-Frame-Options响应头，默认值DENY
	是否允许被iframe访问，DENY， SAMEORIGIN， ALLOW-FROM
	DENY：禁止被iframe
	SAMEORIGIN：允许同源
	ALLOW-FROM：这种不支持配置需要AllowFromStrategy类处理

####HpkpConfig 
	添加Public-Key-Pins或者Public-Key-Pins-Report-Only
	默认无配置，不启用
	https相关的东东

####ContentSecurityPolicyConfig
	添加Content-Security-Policy响应头或者Content-Security-Policy-Report-Only
	与xss漏洞有关，默认无配置不开启

#### ReferrerPolicyConfig 默认无配置，不启用

	Referrer-Policy响应头
	发送referer的策略

#### FeaturePolicyConfig
	Feature-Policy响应头
	默认无配置，不启用
	控制浏览器启用或禁止一些api

***
	springSecurity会根据上面写的规则，添加若干个响应头。
	我遇到的情况是，因为默认启用了FrameOptionsConfig:deny，导致了前端不能访问图片资源了，将这个策略禁用即可


```

//禁用headerFilter 的禁止iframe访问策略
http.headers().frameOptions().disable();

```