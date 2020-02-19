---

layout:     post
title:      "springboot Maven打包"
subtitle:   ""
date:       2020-02-19
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springBoot
- Maven
typora-root-url: ..
---





今天使用`maven`命令打包`springboot`项目成可执行`jar`，总是无法成功。表现为打包的jar包只有项目本身的`Class`，不包含依赖。

但是把项目文件复制到其他目录就能正常使用。后来发现原因是这这样。



项目是一个多Module项目，父pom中这样写的。



```xml
 <build>
        <pluginManagement>
                <plugins>
                    <plugin>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-maven-plugin</artifactId>
                        <version>2.1.8.RELEASE</version>
                        <configuration>
                            <skip>true</skip>
                        </configuration>
                        <executions>
                            <execution>
                                <goals>
                                    <goal>repackage</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
        </pluginManagement>
    </build>
```



我做的子Module是这样写的。

```xml
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```





这样问题就产生了，子项目是继承父pom的，这样子项目没有写版本号，但是我没注意看到父pom有一个

`<skip>true</skip>`，这个选项的意思是忽略`execution`，这样插件就不会执行。

完全没明白同事为什么要这样写，弄了好久才注意到这个选项。



打包命令,顺便跳过测试

```
mvn clean package -Dmaven.test.skip=true
```





