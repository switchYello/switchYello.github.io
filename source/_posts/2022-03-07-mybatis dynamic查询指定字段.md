---
layout:     post
title:      "mybatis dynamic查询指定字段"
subtitle:   ""
date:       2022-03-07
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- mybatis
- mybatis dynamic
typora-root-url: ..
---


# mybatis dynamic查询指定字段

​		使用mybatis dynamic做查询是，通常情况下会查询所有字段，因为多查几个字段对性能影响不会很大。但是有些情况下，查询记录数量较多，多查的字段就有很大的影响了。
此时我们希望只查询指定的字段。



## 第一种写法

一般写法是这样的

```java
testMapper.selectMany(MyBatis3Utils.select(columns,test, dsl->{
            dsl.where(reconSerialNo,SqlBuilder.isEqualTo("xxx"))
        }))
```

我们需要传递三个参数，第一个参数为想要查询的列数组，第二个参数为想要查询的表，第三个参数为SelectDSLCompleter 查询条件。

其原理就是使用Mybatis3Utils构建一个SelectStatementProvider对象。

```java
public static SelectStatementProvider select(BasicColumn[] selectList, SqlTable table,        SelectDSLCompleter completer) {   
    return select(SqlBuilder.select(selectList).from(table), completer);
}
```



## 第二种写法

除此之外，我么还可以像下面这样写：

```java
testMapper.select(dsl->{    
    SqlBuilder.select(columns).from(test).where(reconSerialNo, SqlBuilder.isEqualTo("xxx"))
})
```



​	第二种写法不通过参数传递字段和表名，而是在dsl覆盖掉字段和表名。按照第一种写法查看其源码，SelectDSLCompleter的参数dsl本身就是 `SqlBuilder.select(columns).from(test)`，我们没有使用它，而是在dsl内部重写了，忽略回调传递的参数，从而实现查询指定字段。

