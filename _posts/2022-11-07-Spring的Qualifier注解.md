---
layout:     post
title:      "Spring的Qualifier注解"
subtitle:   ""
date:       2022-11-07
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- Spring
- Qualifier
- Spring注解
typora-root-url: ..
---


# Spring的Qualifier注解



​	spring的依赖注入功能依赖`@Autowired`注解，该注解可以标记在单个字段上，表示注入单个值。也可以标记在list/array/map上，表示注入多个值

​	`@Qualifier`注解是在@Autowired的基础上提供了选择的功能，有如下使用场景

1. 需要注入一个对象，但容器内存在多个相同类型的对象，容器无法选择具体注入哪一个

2. 容器内存在5个相同类型的对象，需要指定注入其中3个到一个List中



### 用法

​	@Qualifier有两种用法，一种直接在需要注入的地方添加@Qualifier注解，一种添加注解时顺便指定值，例如@Qualifier(value='a1')表示筛选名称为a1对象或同样添加@Qualifier(value='a1')的bean。

​	

​	@Qualifier的这两种用法是有区别如下：

如果只添加@Qualifier注解，没有指定值，则查找的依据为是否同样添加@Qualifier。只有添加了@Qualifier注解（ps:注解上不能有值）才能满足筛选条件。

如果添加的@Qualifier(value='a1'),则存在两种类型的bean是符合条件的。

- bean的定义中同样添加了@Qualifier(value='a1')	

- bean的name=a1

  

### 示例

​	 @Qualifier注解没有参数的情况下，会查找bean定义时增加@Qualifier的bean，此时注入的是a2

```java

    static class Test {

        interface A {

        }

        @Component("a1")
        class A1 implements A {

        }

        @Qualifier
        @Component("a2")
        class A2 implements A {

        }

        @Qualifier
        @Autowired
        A a
        
    }
```

​	

​	@Qualifier注解上指定了bean的名字为a1，所以即使a2上存在@Qualifier注解也不会选中，此时注入的bean是a1

```java

    static class Test {

        interface A {

        }

        @Component("a1")
        class A1 implements A {

        }

        @Qualifier
        @Component("a2")
        class A2 implements A {

        }

        @Qualifier("a1")
        @Autowired
        A a
        
    }
```



​	a1 bean满足注入条件，因为@Qualifier上指定了bean的名字是a1，同时a2 bean也满足注入条件，因为他与需要注入的字段上增加的`@Qualifier("a1")`是一样的。此时两个bean都满足条件，启动会报错。

```java
	 static class Test {

        interface A {

        }

        @Component("a1")
        class A1 implements A {

        }

        @Qualifier
        @Component("a1")
        class A2 implements A {

        }

        @Qualifier("a1")
        @Autowired
        A a
        
    }
```



​	注入列表与注入字段一样的，此时注入到list中的对象只有a2，因为它上面添加了@Qualifier注解

```java
    static class Test {

        interface A {

        }

        @Component("a1")
        class A1 implements A {

        }

        @Qualifier
        @Component
        class A2 implements A {

        }

        @Qualifier
        @Autowired
        List<A > a
        
    }
```



​		再看一下最后一种情况，这种情况下两个bean都不会被注入，a1上没有@Qualifier注解不满足条件，a2上的@Qualifier("a2")注解与需要注入的字段上的@Qualifier注解不一致也不满足条件。

```java

    static class Test {

        interface A {

        }

        @Component
        class A1 implements A {

        }

        @Qualifier("a2")
        @Component
        class A2 implements A {

        }

        @Qualifier
        @Autowired
        List<A > a
        
    }
```

