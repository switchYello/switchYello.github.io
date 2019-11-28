---
layout:     post
title:      "springboot+idea 配置热部署"
subtitle:   ""
date:       2019-05-31
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springboot
---


# springboot+idea 配置热部署

***

## 1.maven

```
       <!-- 热部署模块 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>

```


## 2.maven 插件
```
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <fork>true</fork>
                </configuration>
            </plugin>

```


## 3.idea 设置

	ctrl+alt+s 打开idea的设置，
	找到[Build，Execution,Deployment] - > [Complier] -> 打勾[Build project automatically]

## 4.idea设置
	
	ctrl+alt+shift+/, 打开的窗口选择第一项[Registry]
	找到[ compiler.automake.allow.when.app.running ] 后面打勾


## 5.debug模式下启动项目
	
	不出意外，修改类会自动加载，并且速度非常快