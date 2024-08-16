---
layout:     post
title:      "springboot单元测试 ，spring项目单元测试"
subtitle:   ""
date:       2019-05-31
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springboot
---


# springboot单元测试 ，spring项目单元测试

# 1.springboot
## 添加maven


```	
	  <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
```

## 编写一个测试类，加上注解

```java

//启动类所在的class
@SpringBootTest(classes = DemoApplication.class)
@RunWith(SpringRunner.class)            //固定的
public class Test {
	//注入需要用到的类
	@Autowired
	private BingImageMapper bingImageMapper;


	//测试执行的方法
	@org.junit.Test
	public void test() throws IOException {
		//可以正常使用
		bingImageMapper.selectByPrimaryKey(1);
	}
}


```

## 运行 
	直接执行test方法即可

# 2.spring
## 编写测试类继承 `AbstractJUnit4SpringContextTests`，并添加注解，将`web.xml`中配置的的配置文件也写在注解里


```java

@ContextConfiguration({"/applicationContext.xml","/testspring/applicationContext.xml","/dao-applicationContext-db.xml"})
public class PointTest extends AbstractJUnit4SpringContextTests{
	//注入需要用到的类
	@Autowired
	private BingImageMapper bingImageMapper;

	//测试执行的方法
	@org.junit.Test
	public void test() throws IOException {
	//可以正常使用
		bingImageMapper.selectByPrimaryKey(1);
	}
}


```

## 如果有多个配置，全写进去