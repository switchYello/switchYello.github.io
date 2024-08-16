---
layout:     post
title:      "mybatis 找不到statement"
subtitle:   ""
date:       2019-07-30
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - java
---


# mybatis 找不到statement “Mapped Statements collection does not contain value for xx “


### 我发生这样的原因是因为我的扫描配置的是 ‘classpath:mapper/*Mapper.xml’ ,而我后面新建的mapper文件命名是，userMapperEx.xml 导致命名和通配命名不一致，扫描不到。在此记录一下。