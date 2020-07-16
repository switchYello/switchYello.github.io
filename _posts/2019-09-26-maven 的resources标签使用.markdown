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



## 默认情况下的maven资源配置



​		此标签用来控制资源的拷贝，maven对资源的控制是使用`maven-resources`插件完成的。

使用`mvn help:effective-pom`查看实际使用的配置。`resources插件`默认会执行`testResources` 和 `resources` 两个goal

```xml
<plugin>
	<artifactId>maven-resources-plugin</artifactId>
	<version>2.6</version>
	<executions>
	  <execution>
		<id>default-testResources</id>
		<phase>process-test-resources</phase>
		<goals>
		  <goal>testResources</goal>
		</goals>
	  </execution>
	  <execution>
		<id>default-resources</id>
		<phase>process-resources</phase>
		<goals>
		  <goal>resources</goal>
		</goals>
	  </execution>
    </executions>
</plugin>
```



​		resources插件默认执行上面两个goal，这两个步骤分别对测试资源和正式资源进行拷贝。

从哪里拷贝到哪里是由`resources`标签决定。默认`resources`标签是：

```xml
<outputDirectory>C:/ijtest/testmaven/target/classes</outputDirectory>
<testOutputDirectory>C:/ijtest/testmaven/target/test-classes</testOutputDirectory>

<resources>
  <resource>
	<directory>C:/ijtest/testmaven/src/main/resources</directory>
  </resource>
</resources>
<testResources>
  <testResource>
	<directory>C:/ijtest/testmaven/src/test/resources</directory>
  </testResource>
</testResources>
```

​		可以看到默认目标分别是`src/main/resources` 和 `src/test/resources` ， 而输出目标分别是`target/classes` 和`target/test-classes`。



## 自定义Resources标签

​	我们自定义`resources`标签可以覆盖超级pom的默认配置。下面只讲解`resources`标签，`testResources`标签是同理的。

```xml
 	<build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <targetPath>abc</targetPath>
                <includes>
                    <include>application.*</include>
                </includes>
                <filtering>true</filtering>
            </resource>
        </resources>
    </build>

```

#### 这个标签写在`build`标签下，用来管理resources插件工作

+ ### directory

    ​		表示原资源路径，请填写相对路径，相对于pom所在的目录，即如果`pom.xml`在`d:project`文件夹下，默认的相对路径为`d:project/`  ，拼在一起就变成`d:project/src/main/resources`，所以这里填写路径不要以 `/`开头，否则会变成绝对路径。这个选项是必填的。

    ​	

+ ### targetPath

  ​	 	资源拷贝的目标地址，这个路径是选填的。默认输出路径为`target/classes/`，如此处填写`abc`，

  则会将`src/main/resources`下的文件拷贝到`target/classes/abc`文件夹下。不写就是拷贝到`target/classes/`。

  

+  ###    includes 和 excludes。
	
	maven默认会将`directory`下的文件全部拷贝到`targetPath`下，但是受这两个属性控制。
	
	1. 如果存在 `includes` 则只将`includes`指定的文件拷贝，其余不拷贝。
	2. 如果存在`excludes` 则将排除掉指定文件外的剩余文件拷贝。
	3. 如果同时存在，则按照先后顺序过滤。例如先存在`includes`，后存在`excludes`，则先使用`includes`筛选出满足条件的文件，再使用`excludes`对上面筛选出来的文件进行二次筛选，筛选通过的拷贝进项目。
	4. 如果同时存在，且同时操作同一个文件，则以excludes为准。因为无论excludes在前还是在后，都会将文件过滤掉。
	5. 上面的四个规则是对于同一个`resource`标签内的`includes、excludes`来讲的，不是同一个`resource`标签互相不影响。例如下面的例子使用一个`resource`过滤掉所有的配置，使用另一个`resource`添加指定的配置。
	
	
	
	​	有时使用单个`resource`标签可能无法完成想要的行为，可以多个`resource`组合使用如
	
	想要`resources`文件夹下的`application*.properties`文件，根据配置选择打包哪个，可以这样写。
	
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

    ​	这个选项可以是true或 false，表示是否对资源进行过滤，也就是说，可以在配置文件里写一些动态的值，maven会将值替换成真正的值，设成false则不会过滤。

    

    比如下面`properties`文件中的一条记录，maven编译完成后值会被替换成`profile.active`这个变量代表的值。

    ```properties
    spring.profiles.active= @profile.active@
    ```



## 总结

​		需要注意的一点是，如果在自己的pom里配置的resource标签，超级父pom里的resource就会失效了，比如下面这样配置后，只会拷贝`src/main/myImage`里的内容，默认的`src/main/resources`内的内容就不会被拷贝了。

```xml
<resources>
  <resource>
	<directory>src/main/myImage</directory>
  </resource>
</resources>
```

​		

​		可以再添加一个`<resource>src/main/resources</resource>` ，两个`resource`共同配置即可。





