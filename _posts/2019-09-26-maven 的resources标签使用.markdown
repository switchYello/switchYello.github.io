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

```
 	<build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <targetPath>BOOT-INF/classes</targetPath>
                <includes>
                    <include>application.*</include>
                </includes>
            </resource>
        </resources>
    </build>

```


#### 这个标签使用在`build`标签下，表示将什么资源复制到打包的包中去

+ ### directory 
	表示原路径，请填写相对路径，默认为pom所在的路径，即如果pom在`d:project`文件夹下，默认的相对路径为`d:project/`  这里路径不要以 `/`开头，否则会变成绝对路径

+ ### targetPath 
	表示打包成jar或war的路径，相对于压缩包根目录



+  ###    includes 和 excludes，maven默认会将directory下的文件全部拷贝到targetPath下
	
	1. 如果存在 includes 则只将includes指定的文件拷贝，其余不拷贝
	2. 如果存在 excludes 则将排出了指定文件外的全部文件拷贝
	3. 如果同时存在，且同时操作同一个文件，则以excludes为准