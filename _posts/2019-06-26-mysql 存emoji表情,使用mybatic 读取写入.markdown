---
layout:     post
title:      "mysql 存emoji表情,使用mybatic 读取写入"
subtitle:   ""
date:       2019-06-26
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - java
---


# mysql 存emoji表情,使用mybatis 读取写入

	1.首先把mysql对应的行的编码改成utf8mb4,不用改整个表或库,只需改这一行的编码格式就可以了
	我是通过navicat修改的.

	2.数据库连接驱动最低版本 5.1.47

	3.数据库连接url中配置编码为utf-8
	
	4.其他不变,尝试下不出意外就可以了进行插入 读取了