---
layout:     post
title:      "jenkins自动构建项目并发布到maven私服"
subtitle:   ""
date:       2019-10-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - jenkins
---


### jenkins自动构建项目并发布到maven私服

#### 前提条件
	安装好jenkins
	项目maven能正常手动发布
	项目源码从svn或者git管理

#### 达到的目的
	jenkins会轮询svn项目的更改，如果发现项目更改了，则重新编译，并发布到maven私服上

#### 做法

	1.打开jenkins网页，添加自由风格的项目，名字任意如【project1】
	
	2.配置General
		2.1 配置Discard old builds指定保存的记录天数和最大构建个数，自动删除前面的构建，防止占用太多存储空间
		
	3.源码管理我选择subversion，填写url选择密码凭据
		3.1这样每次执行任务时，会将svn的源码下载到该文件夹下，没有该文件夹会自动创建
		
	4.构建触发器选择Poll SCM，定时轮询svn服务器
		4.1其中corn表达式和平时用的稍有不同,点击输入框后面的问号有说明，我这里写 H * * * * 即每分钟检查一次，因为开发阶段更新比较频繁
		
	5.构建步骤选择`Execute windows batch command`即执行window批处理脚本，因为jenkins我是在本机windows上安装的
		5.1使用jenkins调用我们的bat脚本，来进行项目编译发布
		5.2如果我们配置123.bat，默认会调用【jenkins安装目录\workspace\project1\123.bat】，也可以写死绝对路经如【c:123.bat】这样也可以找到
		5.3我将脚本放置在【jenkins安装目录\workspace\autoPublish.bat】所以我配置路径时写的是【../autoPublish.bat】这样就能找到我的脚本了
		5.3脚本执行环境的目录为【jenkind安装目录\workspace\project\】

	6.bat文件中我们可以这样写 
	::=========================
			@echo off
			mvn deploy
	::=========================
	
	7.这里面写的就是普通maven命令，他会自动将项目进行编译，测试，发布到私服，前提条件是已经配置好pom，能手动完成