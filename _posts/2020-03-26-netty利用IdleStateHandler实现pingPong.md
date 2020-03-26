---

layout:     post
title:      "netty利用IdleStateHandler实现pingPong"
subtitle:   ""
date:       2020-03-26
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- netty
typora-root-url: ..
---





`IdleStateHandler`这个类能实现三种工作模式 `长时间未读` `长时间未写` `长时间未读写`，三种模式可以同时工作

看一下它的构造方法

```java
   public IdleStateHandler(
            int readerIdleTimeSeconds,
            int writerIdleTimeSeconds,
            int allIdleTimeSeconds) {

        this(readerIdleTimeSeconds, writerIdleTimeSeconds, allIdleTimeSeconds,
             TimeUnit.SECONDS);
    }
```



分别传递读超时，写超时，读写超时 的时间，当某种模式超时时会回掉下面这个方法，将事件传递过来。我们判断这个方法里的事件来判断是什么超时了。如果参数传0则表示不检测这一项。

```java
 protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
        ctx.fireUserEventTriggered(evt);
    }
```





### 改造成PingPongHandler

`PingPongHandler` 写超时时会发送`Ping`，如果一定时间没接收到`Pong`，则主动断开连接。



接收到的`Pong`除了刷新最后读时间外没有其他用处，在`PingPongHandler`后面添加一个`SimpleChannelInboundHandler`来忽略所有的Pong



```java
public class PingPongHandler extends IdleStateHandler {

    private static Logger log = LoggerFactory.getLogger(PingPongHandler.class);
    private static int writeTimeOut = 10;
    private static int readTimeout = writeTimeOut * 4 + 2;

    public PingPongHandler() {
        super(readTimeout, writeTimeOut, 0);
    }

    /**
     * PingPongHandler这个类，利用了IdleStateHandler定时发送ping
     * 但是收到的Pong如果不处理会被继续向下传播，这里添加一个SimpleChannelInboundHandler<Pong>在此handler后面，来丢弃所有的Pong，并且能刷新最后读取时间。
     */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        super.handlerAdded(ctx);
        ctx.pipeline().addAfter(ctx.name(), null, new SimpleChannelInboundHandler<Pong>() {
            @Override
            protected void channelRead0(ChannelHandlerContext ctx, Pong msg) {
                log.debug("收到Pong");
            }
        });
    }

    /*
     * 长时间没写（写超时），则主动发送ping
     * 长时间没读（读超时），则断开连接
     * */
    @Override
    protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) {
        if (evt.state() == IdleState.WRITER_IDLE) {
            ctx.writeAndFlush(new Ping()).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
        } else {
            log.debug("读超时断开连接：{}", evt);
            ctx.flush().close();
        }
    }

}
```







