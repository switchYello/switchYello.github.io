---

layout:     post
title:      "spring Xml配置枚举"
subtitle:   ""
date:       2020-05-08
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring Xml配置枚举
---



​		在spring的配置文件中配置bean很简单，但某个bean中字段是枚举类型，如何注入呢。



```xml

    <bean id="jacksonHttpMessageConverter"
          class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
        <property name="objectMapper">
            <bean class="com.fasterxml.jackson.databind.ObjectMapper">
                <!--值为null时不返回-->
                <property name="serializationInclusion">
                    <util:constant static-field="com.fasterxml.jackson.annotation.JsonInclude.Include.NON_NULL"/>
                </property>
            </bean>
        </property>
    </bean>

```



上面的`com.fasterxml.jackson.annotation.JsonInclude.Include.NON_NULL`就是枚举类型，可以使用这种方式注入。