---

layout:     post
title:      "maven打包成minijar"
subtitle:   ""
date:       2020-03-21
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- maven
- maven打包miniJar
typora-root-url: ..
---





本文演示使用`maven`的`maven-shade-plugin`插件实现打包成`minijar`功能，并解决缺少依赖问题。





`maven`打包成可执行`jar`包时需要将依赖的`jar`包也一同打包到`jar`包内，最终可能导致得到的可执行`jar`体积太大，

但是第三方`jar`的部分`class`可能是从未使用到的，即使移除也不会影响程序运行，这样可以大大减少打包的体积。





插件`maven-shade-plugin`为我们提供了这样的功能，他会自动分析`jar`包内的依赖，将不需要的`class`移除。

```xml
    <plugin>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.2.1</version>
        <configuration>
            <minimizeJar>true</minimizeJar>
        </configuration>
        <executions>
            <execution>
                <phase>package</phase>
                <goals>
                    <goal>shade</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
```



上面配置插件，并挂钩到`package`步骤中，执行`shade`。配置`minimizeJar`为`true`即可。





## 部分依赖问题

上面很简单能实现`minijar`，但是插件在分析依赖时不能分析到反射获取的类，这会导致程序运行时反射获取类失败，所以需要手动处理这样的类。



下面我们在使用`logback`时，因为配置是写在配置文件里的，所以部分类会被插件丢弃掉导致`class Not Find`。

可以配置`filter`节点手动包含类，规则是。

```xml
    <filter>
        <artifact>artifactId:artifactId</artifact>
            <includes>
                <include>**</include>
           </includes>
    </filter>
```



示例，包含`logback-core`的所有类：

```xml
    <plugin>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.2.1</version>
        <configuration>
            <minimizeJar>true</minimizeJar>
            <filters>
                <filter>
                    <artifact>ch.qos.logback:logback-core</artifact>
                    <includes>
                        <include>**</include>
                    </includes>
                </filter>
            </filters>
        </configuration>
        <executions>
            <execution>
                <phase>package</phase>
                <goals>
                    <goal>shade</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
```





​	要注意的一点是，如果配置了`filter`，该`artifact`就不会自动处理依赖了，插件只包含`include`内手动指定的类，没指定的全部不会包含。我这里使用了`**` 表示全部包含进来。



