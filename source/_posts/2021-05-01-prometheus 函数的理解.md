---
layout:     post
title:      "prometheus函数的理解"
subtitle:   ""
date:       2021-05-01
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- prometheus
- grafana
typora-root-url: ..
---



# prometheus函数的理解


## 最近在研究prometheus，花了一天时间理解了几个主要函数的原理，现在记录以下，给以后的自己看



### rate()函数
这个函数接受一个范围counter序列，他能返回每秒钟的qps，因为counter是单调递增的，我才他是将最后一个记录的值减去第一个记录的值，再除以时间段的总秒数
得到一个浮点型数字，表示每秒增长的数量。这个函数只能返回每秒的数量，如果需要每分钟的可以自己乘以60，因为是浮点类型的，所以精度应该还可以接受。

与之对应的还有一个increase()函数，他能求出参数指定的counter序列的首尾数值只差，但他其实是先计算出rate(),再乘以时间段的秒数得出来的。
相当于给rate()函数包一层。官方推荐得到的值只用于人工查看时方便阅读使用。




### sum()函数

他的用法如下
> sum by (label_a,label_b) (query)
  
如果query这个查询返回的是多条序列，他能把label_a相同，且label_b也相同的序列点相加形成一个新的点，
如我们在两台服务器上都能提供同一个序列，使用sum函数按照相同label聚合后，将多台服务器上的只相加，产生一个新的序列。

如使用A服务器产生一个qps的折线图，B服务器也产生一个qps的折线图，按照同一个label聚合后将qps值叠加，看起来更清晰。

使用sum函数计算合并histogram时一般要带上 le 做参数，表示le相同才聚合，因为我们不能将不同le的聚合在一起，那样就没意义了。


### histogram_quantile() 函数

> histogram_quantile(q,v)

q是一个 0-1之间的值，v是一个瞬时向量，且v是一个histogram，他带有一个le的标签，此函数能按照直方图的数值和数量大概猜测出耗时，
但是一般使用它时，是不可能将历史所有数据都放在一起计算的，我们要指定时间范围内的数据计算才准确。

所以一般用法如下面这样的

> histogram_quantile(0.9,rate(Request_Count[5m]))

他的原理是先将最近五分钟的数据拿出来，计算出每秒钟的平均次数，其实就是一秒内的数量，使用一秒内的增长值作为直方图，用于计算响应时间。

我考虑过为什么可以拿一秒钟的次数而不是拿五分钟或十分钟的差值做计算，后来想既然rate()函数也是根据指定范围内的差值除以时间秒数得到的，
且结果是浮点小数，只要再乘以时间秒数 （相当于是increase()），就能得到原来的五分钟的时间差值了，所以用一秒钟的数据也不会导致精度损失。



### 关于 瞬时向量和区间向量

有点类似于python里面的数组切片，
query 表示一个瞬时向量
query[1m] 表示一个当前时间一分钟前到现在这一分钟长度的区间向量。




### end

[grafana文档](https://grafana.com/docs/grafana/latest/datasources/prometheus/)


[prometheus文档](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile)




## 补充 grafana一些细节

### 定义变量

定义常量类型的变量，将Type设为Constant
常用的变量类型是Query表示从prometheus中查询结果作为变量，还可以从查询的结果中使用正则表达式切割出想要的字符串作为变量

如下

__name__ 表示指标的名称，这里使用正则表达式匹配满足名称为`PaymentSubmitChannel[0-9a-zA-Z]*_bucket` 且job等于变量job的指标。
再使用正则表达式，将中间的变量部分分组出来
这样出来的结果就是我们想要的中间变化的部分了。

Gragana的正则表达式写法，与js正则相似，需要使用两个斜线将正则表达式包起来

Queyr : {__name__ =~ "PaymentSubmitChannel[0-9a-zA-Z]*_bucket",job="$job" }
Regex : /PaymentSubmitChannel([0-9a-zA-Z]*)_bucket/




### 使用变量
使用变量推荐下面两种方式
{job = "$job}  或者 {job = "${job}}
前一种方式不能将表达式放在字符串中间，后一种方式可以放在字符串中间

如下，将变量放置在PaymentSubmitChannel${paycode}_bucket 字符串里
rate(PaymentSubmitChannel${paycode}_bucket{job = "$job",env="$env"}[1m]))














