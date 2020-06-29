---

layout:     post
title:      "jekyll制作sitemap"
subtitle:   ""
date:       2019-12-19
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- jekyll
- gitpage
typora-root-url: ..
---

# jekyll制作sitemap



jekyll提供制作sitemap的插件，如果不通过插件制作的话可以写一个`sitemap.xml`模板，模板内遍历文章列表，生成sitemap。

以下是我的模板，根据参考文章中改的。

将下面代码复制到项目根目录，命名为`sitemap.xml`，这样生成的sitemap就会在站点根目录下即可。

需要`_config.yml`文件内有一个`sitemapUrl`变量来作为sitemap的url，拼接在文章相对地址前面。

上面一部分循环所有文章，下面一部分循环所有除文章外的page，我把带xml的排出了，因为有一个rss订阅相关的`feed.xml`不希望显示在站点地图里。



![image-20191219120135944](/img/in/2019-12-19-jekyll制作sitemap/image-20191219120135944.png)





### 参考资料

[谷歌给的站点地图指南，站点地图各项的意义](https://support.google.com/webmasters/answer/183668?hl=zh-Hans)

[某人写的生成站点地图模板，我是根据这个改的](http://davidensinger.com/2013/11/building-a-better-sitemap-xml-with-jekyll/)

[sitemap.xml的写法](http://blog.sina.com.cn/s/blog_6a3c6f810100zq72.html)