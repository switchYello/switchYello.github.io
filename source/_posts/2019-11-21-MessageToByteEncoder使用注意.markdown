---
layout:     post
title:      "MessageToByteEncoder使用注意"
subtitle:   ""
date:       2019-11-21
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - netty
---


## MessageToByteEncoder使用注意

	这个encoder会将message解析成byteBuffer，但要注意的一点是如果传入encode方法的message没有被消耗会没有被完全消耗，
	剩余没消耗完成的部分会被丢弃掉的从而导致encode出来的数据不全。

#### ByteToMessageCodec
​	这个双向codec内部使用的也是MessageToByteEncoder，也有同样的问题

