---
layout: post
title: 花一天时间折腾下gitpage，将博客迁移到gitpage上
subtitle:   ""
date:   2019-11-28
author:	"hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - gitpage
    - jekyll
    
---



### 花一天时间折腾下gitpage，将博客迁移到gitpage上


今天花点时间将博客迁移到gitpage上，下面总结下过程

* TOC
{:toc}


#### 1.创建仓库

这个没什么难度，创建一个[你的名字.github.io]的空白仓库即可

#### 2.开启gitpage

在该仓库的设置里，开启gitpage，访问网址[你的名字.github.io]即可访问到空白的页面

#### 3.搜索下载喜欢的jekyll主题

一顿百度，最后在知乎上找到一个好看的主题 [帖子链接](https://www.zhihu.com/question/20223939),于是下载之

#### 4.修改主题

解压出来的文件目录如下:

![文件目录](/img/in/花一天时间折腾下gitpage，将博客迁移到gitpage上/1.png)

	includes 模板html
	layouts  文章样式模板html
	poists	markdown 源文章
	site	jkeyll自动生成的，可删除
	css 	样式
	fonts 	字体
	img  	各种图片
	js		js
	less	样式文件
	pwa		
	config.yml 主配置
等等 [仓库地址](https://github.com/Huxpro/huxpro.github.io) readme里面有详细解释

#### 5.将自己的文章转成主题能使用的形式
> 也就仿照着这个头部将信息填写进去，，上下各用三个横线包住。

![文章格式](/img/in/花一天时间折腾下gitpage，将博客迁移到gitpage上/2.png)

可以看出文章文件名命名规则为 
	年-月-日-标题
文章内部最前面个，需要加入下面这种格式的来识别标题等

	---
	layout:     post    #选择那个主题 layouts文件夹里的
	title:      "你好，世界"  #标题
	subtitle:   ""			#副标题
	date:       2019-05-26  #日期
	author:     "hcy"		#作者
	header-img: "img/gene-head-img.jpg" #文章头图
	tags:
	    - diary			#打标签
	---
	
	
	今天是 2019-05-26	#正文


#### 6.本地运行预览
需要在本地搭建jekyll环境，而jekyll是用ruby开发的

##### 下载ruby及安装
这个软件对window兼容不太好，教程参考这个[window上运行jekyll](http://jekyll-windows.juthilo.com/5-running-jekyll/)进行安装

文件所在的目录下，调用`jekyll serve`命令测试运行，如果没报错的话，控制台上会显示打开127.0.0.1:4000 就可以访问啦
如果报错的话，可以看看下面的内容，或者根据错误信息查询，记得主配置文件config.yml里面的东西要好好看一下，一条一条的看


#### 7.遇到的问题
##### jekyll中文文件名本地预览问题，中文的文件名本地无法打开，会报404
解决办法[jekyll中文文件名本地预览问题](http://kael-aiur.com/%E5%85%A5%E9%97%A8%E6%8C%87%E5%BC%95/jekyll%E4%B8%AD%E6%96%87%E6%96%87%E4%BB%B6%E5%90%8D%E6%9C%AC%E5%9C%B0%E9%A2%84%E8%A7%88%E9%97%AE%E9%A2%98.html)如下，

修改安装目录\Ruby22-x64\lib\ruby\2.2.0\webrick\httpservlet下的filehandler.rb文件，建议先备份。

找到下列两处，添加一句（ + 的一行为添加部分）
	
```
path = req.path_info.dup.force_encoding(Encoding.find("filesystem"))
+ path.force_encoding("UTF-8") # 加入编码
if trailing_pathsep?(req.path_info)
```

```
break if base == "/"
+ base.force_encoding("UTF-8") # 加入編碼
break unless File.directory?(File.expand_path(res.filename + base))
```

然后重启服务器即可jekyll clean && jekyll s

##### 报错

	Dependency Error: Yikes! It looks like you don’t have jekyll-paginate or one of its dependencies installed. 
	In order to use Jekyll as currently configured, you’ll need to install this gem. The full error message from Ruby is: ‘cannot load such file – jekyll-paginate’ If you run into trouble,
 	you can find helpful resources at Getting Help

运行`gem install jekyll-paginate`安装jekyll-paginate即可


#### 8.上传到github

	直接git绑定远程仓库，上传等30s左右就能打开了

	git remote add origin Urlxxxxxx
	git push origin master
