---
layout:     post
title:      "Flux parallel操作符的用法"
subtitle:   ""
date:       2022-01-17
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- flux
- parallel
- reactor
typora-root-url: ..
---


# Flux parallel操作符

## 1. parallel操作符
​	parallel操作符执行过后，会生成一个新的flux 即parallelFLux。他能够接受多个订阅者，他会从原始flux中请求perfetch个元素，将上游传递push过来的元素依次按顺序分配给子订阅者。

​	分配上游元素过程是：依次分配给下游每个订阅者，当某个子订阅者请求数已经满足时，将跳过此订阅者继续向下一个子订阅者分配。

​	他有连个参数 parallelism 和 prefetch，第一个参数表示划分子流的数量默认等于cpu核心数，prefetch表示向原始流预取的数量。

## 2. parallel操作符后接map操作符
  	map操作符会将真实订阅者生成代理，他将上游onNext的值使用map操作符进行转换。在parallelFlux场景下，它会将每个子订阅者包装起来进行转换。


## 3. parallel后面可以接Sequential操作符
​	可以将parallelFlux转成普通flux，他的原理是创建parallel需要的子订阅者，使用子订阅者订阅parallelFlux，然后将数据依次从每个子订阅者中获取到数据，向下游发送。
​	在sequential操作执行时，数据可能来自多个线程，这里采用的是哪个线程先到则负责将其他子订阅者队列中的值向下发，没获取到锁的线程存入队列直接退出。允许传递参数，表示每个子订阅者身上能存储的元素最大数。
（这里想到支持并发的websocket也是这个原理）


## 4.parallel后接runOn操作符
​	本质上是包装每个子订阅者，将接受到的数据存入子订阅者的queue中，然后使用线程池处理queue内的数据。
处理的逻辑也是和上面Sequential类似，使用AtomicInteger限制每个子订阅者同时只会有一个线程在执行，如果当前已经存在一个线程在执行，后来的数据存入queue后会直接退出。

​	它还可以传递第二个参数表示每个子订阅者预取的数量，这里为什么要预取呢，是因为如果下游订阅者请求了非常大的数量，而runOn将任务分配到线程池中本身是非堵塞的，也就是任务全部堆积在线程池中，可能导致内存溢出。所以使用预取参数分批次满足下游的请求数量，默认值。




