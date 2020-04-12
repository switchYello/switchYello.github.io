---
layout:     post
title:      "mysql 存emoji表情,使用mybatic 读取写入"
subtitle:   ""
date:       2019-06-26
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - java
    - mybatis
    - 技巧
---



# mysql 存emoji表情,使用mybatis 读取写入



## 设置mysql编码

​        首先把`mysql`对应的行的编码改成`utf8mb4`,不用改整个表或库,只需改这一行的编码格式就可以了
我是通过`navicat`修改的.



## 驱动版本

​         数据库连接驱动最低版本 5.1.47，经测试低于这个版本，不能成功。



## 设置数据库连接

​        连接内设置编码

```properties
jdbc:mysql://127.0.0.1/testuseUnicode=true&characterEncoding=UTF8&mysqlEncoding=utf8
```



## 配置连接池参数

​        每个连接吃都有init参数,Springboot + hikari可以这么配置

```properties
spring.datasource.hikari.connection-init-sql=set names utf8mb4
```



不出意外应该可以读写emoji表情了。