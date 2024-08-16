---

layout:     post
title:      "80端口被Microsoft-HTTPAPI2.0"
subtitle:   ""
date:       2020-04-23
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- window
- Microsoft-HTTPAPI2.0
- 80端口被占用
typora-root-url: ..
---



​         在电脑上开反代，启动`nginx`提示`80`端口被占用，浏览器访问`localhost`提示`Microsoft-HTTPAPI2.0`。

不但被占用了80端口，而且是开机自启的，原因是以前在电脑上弄`iis`玩，后来也没在意。升级window系统新版本后，这个变成自动启动了，想办法停止它就行了。

做一下两步即可

1. 管理员`CMD`输入命令 `iisreset /stop` 停止iis。
2. window菜单栏搜索`iis`关键字，打开`iis管理器`，将里面设置的网站删除。

