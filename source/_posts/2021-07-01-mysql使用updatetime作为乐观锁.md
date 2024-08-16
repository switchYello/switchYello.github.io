---
layout:     post
title:      "mysql使用updatetime作为乐观锁"
subtitle:   ""
date:       2021-07-01
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- mysql
- bug
- 乐观锁
typora-root-url: ..
---



# mysql使用updatetime作为乐观锁

本文记录一下，我是用mysql的updatetime字段作为乐观锁版本号遇到的问题。


## 乐观锁
首先标准的乐观锁，应该存在一个version字段，每次更新时人工自增此字段，但是这样每次更新时都要多维护一个字段很麻烦。
所以我使用数据库中的update_time字段作为乐观锁的version使用，将update_time字段设为每次有更新时自动刷新，使用此字段当作version使用，免去了每次自己维度版本号的问题。

如下面sql:
```sql
`update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间'
```

使用乐观锁时是这样的:
```sql
update table set  field = 'field' where id = 1 and update_time = update_time
```


## 精度问题
但是这又引入了新的问题，默认情况下datetime的精度是秒级的，如果同一秒的两次更新，乐观锁是不生效的。


为了解决这个问题，我将update_time字段的精度设置为小数点后6位，缩小精度后，乐观锁只有在同时0.000001秒内同时执行才会失效。
这样改动后在我需要的场景下，是可以满足要求的，一方面是因为与数据库的网络交互本身就比0.000001秒长，另一方面我的场景不需要乐观锁百分百生效

如下是我使用了的sql更新表结构。
```sql
`update_time` datetime(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '更新时间'
```


改动后手动更新一条数据，发现update_time字段果然保留了小数点后6位，于是我将代码部署到测试环境进行测试。


## 踩坑

部署测试环境开始测试后，发现我的更新语句失效了，更新条数是0条。于是我在本地进行测试。

```java
//使用id = 1 从数据库中查询一条记录
Order order = queryById(1)

//更新id = 1的记录，并在where条件中加上update_time
int updateCount = updateOrder(1,order.getUpdateTime)

//更新的条数，竟然是0条
assert updateCount == 0

```

查询数据，然后再更新竟然不行，更新失败了？？？

## 解释
其实原因是，java里面的Date存储的是Long型的毫秒值，精度只到毫秒单位，而数据库里面的update_time是小数点后6位，已经到纳秒级别了，所以从数据库查询数据时，毫秒后面的被忽略了。
所以更新的时候才导致无法更新。


解决这个问题的话，将数据库update_time精度设为小数点后3位即可。毫秒级别也已经很小了，毕竟一次数据库网络传输也要1-20毫秒

```sql
`update_time` datetime(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间'
```
