---
layout:     post
title:      "free-mybatis-plugin的使用方法"
subtitle:   "idea,mybatis generator gui,代码生成，mybatis分页"
date:       2019-12-11
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- idea
- mybatis
typora-root-url: ..
---



## free-mybatis-plugin的使用方法

* TOC
{:toc}



​	作为java界开发必备神器idea，其功能强大，提供强大的插件系统，如果使用当前主流的ssm三大框架，一定要安装free-mybatis-plugin。其大大方便了我们使用mybatis。

该插件不仅仅在mapper接口和mapper.xml文件上提供跳转按钮，还内置了generator gui界面，下面看看如何使用。



### 插件功能

插件一共提供了四个功能:

- 生成mapper xml文件
- 快速从代码跳转到mapper及从mapper返回代码
- mybatis自动补全及语法错误提示
- 集成mybatis generator gui界面



![image-20191211114318462](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211114318462.png)



#### 功能1：快速从代码跳转到mapper及从mapper返回代码

​	这是我们最熟悉的功能，他会在我们的mapper.xml文件的select，delete，update等语句左边提供一个箭头

![image-20191211114749395](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211114749395.png)

点击它就能跳转到与`namespace`对应的mapper接口的相应方法上，方便我们快速定位。



#### 功能2：mybatis自动补全及语法错误提示

​	我们写sql时，弹出的代码提示就是它做的，也是十分的好用。

![image-20191211115058016](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211115058016.png)

#### 功能3，4：集成了`mybatis generator gui`，自动生成代码但比原生的`generator`更好用

​	idea内置了管理数据库的工具，此插件配合数据库管理工具，可以获取到数据库的连接，表，字段等信息，通过这些信息，此插件就能generator 出对应的mapper，dao，entity等。

- 首先打开idea的 `database`窗口，此窗口一班在最右边，和maven同列。

![image-20191211115724055](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211115724055.png)



点击maven上面的`database`按钮会弹出此窗口，点击左边箭头指示的`加号`选择好数据库类型，数据库驱动，输入完账号密码，连接上数据库，下面的schemas就会显示选中的库。我这里事先连上了，可见名称为`test`的库内有一张名称为`troows`的表，表内字段也都显示在上面。



- 在表上右键弹出菜单

![image-20191211120105680](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211120105680.png)

这里第一项`mybatis-generator`则为gui的生成器，下面的选项还有查看建标语句等等功能，这里不多做介绍了，我们选择第一项`mybatis-generator`。弹出：

<img src="/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211120307488.png" alt="image-20191211120307488"  />



第一行选项 `table setting` 自动填充了表名，主键字段，项目文件夹

第二行`model setting`自动填充了生成的实体类文件名，包名，和路径

第三行 `dao setting`自动填充了生成的mapper接口文件名，包名，文件路径

第四行`xml mapper setting`用来设置生成的xml文件路径

第五行`options`是生成的选项，一会一个一个来看



- 我们将路径包名改成自己需要的样子，建议不要选择和项目相同的路径或包名，不然后面生成把前面覆盖了不好把控，可以生成在其他地方，然后复制过来。我将文件生成在java1文件夹下，这样不会干扰本身的程序

    ![image-20191211121018781](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211121018781.png)

点击确定，项目文件夹下生成了如下文件

![image-20191211121202896](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211121202896.png)





### option中配置选项

![image-20191211121332700](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211121332700.png)

#### 1. 分页

单独勾选此选项是没效果的，还需要勾选第五列第二个，生成 example来配合使用才行

查看生成的example中，多了如下代码，可以设置limit和offset

![image-20191211121609488](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211121609488.png)

生成的mapper中，selectByExample方法中多了分页查询的逻辑

![image-20191211121701330](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211121701330.png)



***

#### 2.实体注释

​	勾选这个选项，会在生成的实体类上添加数据库建表时添加的注释，但经过测试，不勾选也能生成注释

![image-20191211121833495](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211121833495.png)

***

#### 3.Overwrite-Xml , Overwrite-Java

​	这两个选项，是生成的代码会覆盖掉原来的代码

***

#### 4.toString/hashcode/equals

​	会在实体类中自动生成上面三个方法

***

#### 5.User-Schema

​	按字面意思是使用数据库名的前缀，我没发现使用这个哪里改变了。

***

#### 6.Add-ForUpdate

这个选项会在example中增加字段

![image-20191211122359965](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211122359965.png)

sql语句中的 selectByExample，会判断该字段，是否加锁查询

![image-20191211122450602](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211122450602.png)

***

#### 7.Repository-annotation

在生成的mapper接口上，添加@repository注解

![image-20191211122608857](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211122608857.png)

***

#### 8.parent-interface

这个会给mapper接口上生成公共的泛型父类，这样TroowMapper内那些增删改查的接口就转移到父类里面去了。看起来清爽些。

![image-20191211122902877](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211122902877.png)

![image-20191211122914657](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211122914657.png)

***

#### 9.jsr310:data and time api

勾选这个，生成实体类中的Data类型会被LocalDateTime代替。

***

#### 10.jpa-Annotation

如图，添加jpa的注解

![image-20191211123320861](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211123320861.png)

***

#### 11.Actual-Column

一般数据库字段都是用下划线命名，java实体类都是用驼峰法命名，生成代码时会从下划线转成驼峰法，勾选这个，生成的实体类会和数据库字段保持一致，保持下划线命名的方式。

***

#### 12.Use-Alias

 这个很有用，勾选这个，查询sql时，会将查询结果起别名，别名为 `表名_字段名`，这有很大好处，关联查询时，就不会出现字段同名问题了。

![image-20191211123744202](/img/in/2019-12-11-free-mybatis-plugin%E7%9A%84%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/image-20191211123744202.png)

***

#### 13.Use-Example

​	上面讲过了，勾选这个才会生成example，且需要example才能完成的功能，也要勾选这个

***

#### 14.Use-Lombox

​	生成的实体类上自动加上Lombox的注解，这样就不会自动生成 get set方法了

***



### 默认配置

​	上面勾选选项，每张表都要重选一次很麻烦，在`file->settings->tools->mybatis generator setting`选项里面，配置的设置，下次使用时会默认选上的，可以设置自己习惯的选项在这上面即可。



### 总结

​	以上就是free-mybatis-plugin插件的用法了。这个插件真的挺强大的，启用别名查询，添加分页，添加注释，添加jpa，都是挺有用的，节省我们手动处理的时间。

​	

