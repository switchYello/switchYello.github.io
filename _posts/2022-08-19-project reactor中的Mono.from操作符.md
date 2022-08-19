---
layout:     post
title:      "project reactor中的Mono.from操作符"
subtitle:   ""
date:       2022-08-19
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- reactor
- Mono.fromCallable
typora-root-url: ..
---


# project reactor中的Mono.from操作符



​	reactor项目中经常会用到Mono.from操作符，主要包含下面四个，他们的返回值如下

-     Mono.fromFuture();  		--> MonoCompletionStage
-     Mono.fromCallable();       -->MonoCallable
-     Mono.fromRunnable();    -->MonoRunnable
-     Mono.fromSupplier();      -->MonoSupplier



​		这几个类的逻辑非常相似，他们有一个共同特点，就是如果supplier返回null时，不会下下游传递，而是直接complete。

```java
	public void subscribe(CoreSubscriber<? super T> actual) {
		Operators.MonoSubscriber<T, T>
				sds = new Operators.MonoSubscriber<>(actual);

		actual.onSubscribe(sds);

		if (sds.isCancelled()) {
			return;
		}

		try {
            //获取真实值，如果值为null则直接完成，否则将值传递给下游后完成
			T t = supplier.get();
			if (t == null) {
				sds.onComplete();
			}
			else {
				sds.complete(t);
			}
		}
		catch (Throwable e) {
			actual.onError(Operators.onOperatorError(e, actual.currentContext()));
		}
	}
```



​	这个特点很有用，特别是搭配Flux嵌套Mono时，可以不用考虑null值的情况。因为Flux是不允许null值的











​		











