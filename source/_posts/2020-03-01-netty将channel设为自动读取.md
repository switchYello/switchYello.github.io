---

layout:     post
title:      "netty将channel设为自动读取"
subtitle:   ""
date:       2020-03-01
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- netty
- java
typora-root-url: ..

---













​	

​	是这样的一个小技巧，这是我在写ss时学到的，有时候要实现私有协议需要将netty设为非自动读取，等前面工作处理好了在转变成自动读取。

​	要实现channel数据的转发工作，第二个channel未创建完成时是不能读取第一个channel内数据的，所以我们经常将第一个channel设为不自动读取。



先前我看netty例子里有这样的写法，转换数据

```java
            out.writeAndFlush(msg).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) {
                    if (future.isSuccess()) {
                        ctx.read();
                    } else {
                        ctx.close();
                    }
                }
            });
```



他的意思就是将msg写入out里面，然后添加监听器，写完后再次从输入流中读取。





后来我又想到另一个效率高的写法,就是这样，将不自动读的输入流重新设为自动读，这样就不用加监听器了：

```java
in.channel().config().setAutoRead(true);
```

