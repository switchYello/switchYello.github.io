---
layout:     post
title:      "maven 的resources标签使用"
subtitle:   ""
date:       2019-09-26
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - maven
---


# maven 的resources标签使用

```xml
 	<build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <targetPath>BOOT-INF/classes</targetPath>
                <includes>
                    <include>application.*</include>
                </includes>
                <filtering>true</filtering>
            </resource>
        </resources>
    </build>

```


#### 这个标签使用在`build`标签下，表示将什么资源复制到打包的包中去

+ ### directory
	

表示原路径，请填写相对路径，默认为pom所在的路径，即如果pom在`d:project`文件夹下，默认的相对路径为`d:project/`  ，拼在一起就变成`d:project/src/main/resources`,所以这里填写路径不要以 `/`开头，否则会变成绝对路径
	
+ ### targetPath
	
	表示打包成jar或war的路径，相对于压缩包根目录，我在打包Springboot似乎可以不用写



+  ###    includes 和 excludes。
	
	maven默认会将directory下的文件全部拷贝到targetPath下，但是受这两个属性控制。
	
	1. 如果存在 includes 则只将includes指定的文件拷贝，其余不拷贝
	2. 如果存在 excludes 则将排出了指定文件外的全部文件拷贝
	3. 如果同时存在，且同时操作同一个文件，则以excludes为准
	
	
	
	有时使用单个`resource`标签可能无法完成想要的行为，可以多个resource可以组合使用如
	
	想要resources文件夹下的`application*.properties`文件,根据配置选择打包哪个，可以这样写。
	
	```xml
	       <resources>
	           //这个resource排除掉所有application*.properties
	            <resource>
	                <directory>src/main/resources</directory>
	                <excludes>
	                    <exclude>application*.properties</exclude>
	                </excludes>
	            </resource>
	           //这个resource添加了application.properties 和 application-${profile.active}.properties
	            <resource>
	                <directory>src/main/resources</directory>
	                <includes>
	                    <include>application.properties</include>
	                    <include>application-${profile.active}.properties</include>
	                </includes>
	            </resource>
	        </resources>
	```
	





+  ### filtering

    ​	这个选项可以是true 或 false，表示是否对资源进行过滤，也就是说，可以在配置文件里写一些动态的值，maven会将值替换成真正的值，设成false则不会过滤。

    

    比如下面`properties`文件中的一条记录，maven编译完成后值会被替换成`profile.active`这个变量代表的值。

    ```properties
    spring.profiles.active= @profile.active@
    ```

