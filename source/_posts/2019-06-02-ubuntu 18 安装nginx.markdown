---
layout:     post
title:      "ubuntu 18 安装nginx"
subtitle:   ""
date:       2019-06-02
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - nginx
---


# ubuntu 18 安装nginx

## 安装nginx依赖

### 安装gcc
	
	apt-get install build-essential
	apt-get install libtool

### 安装pcre依赖库
	
	sudo apt-get update
	sudo apt-get install libpcre3 libpcre3-dev

### 安装zlib依赖库

	apt-get install zlib1g-dev
	
### 安装ssl依赖库

	apt-get install openssl
	
### 安装nginx

	#下载nginx
	wget http://nginx.org/download/nginx-1.17.0.tar.gz
	
	#解压nginx （这个解压出来的文件夹，安装完成后也不要删除，以后再想配置还能用）
	tar -zxvf nginx-1.17.0.tar.gz
	
	#进入解压目录
	cd nginx-1.17.0
	
	#配置
	./configure --prefix=/usr/local/nginx
	
	
	#编辑nginx( 这里只是编译还没有安装)
	make
	
	注意：这里可能会报错，提示“pcre.h No such file or directory”,具体详见：http://stackoverflow.com/questions/22555561/error-building-fatal-error-pcre-h-no-such-file-or-directory
	需要安装 libpcre3-dev,命令为：sudo apt-get install libpcre3-dev
	
	#安装nginx（上面配置时，指定了安装位置）
	sudo make install
	
	启动nginx：
	sudo /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
	
	
## nginx 常用命令

	启动
	/usr/local/nginx/sbin/nginx
	./sbin/nginx　
	
	停止
	./sbin/nginx -s stop
	
	重加载配置文件
	./sbin/nginx -s reload
	
	查看版本，和编译配置信息
	nginx -V
	
## 添加到环境变量

	在/etc/profile 中加入：
	export NGINX_HOME=/usr/local/nginx
	export PATH=$PATH:$NGINX_HOME/sbin	
	保存，
	执行 source /etc/profile ，使配置文件生效
	
## 转载自--> [这篇博客]("https://www.cnblogs.com/piscesLoveCc/p/5794926.html")