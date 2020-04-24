---

layout:     post
title:      "聊一聊ChannelHandler里面方法被调用时机"
subtitle:   ""
date:       2020-04-24
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- netty
- ChannelHandler
typora-root-url: ..
---



​         本文聊一聊`netty`的`ChannelHandler`里面方法被调用的时机，这里面的方法大部分都是被netty回掉的，而不是我们主动调用。

​	     下面的分析都是基于`netty4`来分析，`netty`将`ChannelHandler`分为的`in`和`out`两部分。下面看他们的方法签名。



# 方法签名

## ChannelHandler：

![image-20200424133432059](/img/in/2020-04-24-%E8%81%8A%E4%B8%80%E8%81%8AChannelHandler%E9%87%8C%E9%9D%A2%E6%96%B9%E6%B3%95%E8%A2%AB%E8%B0%83%E7%94%A8%E6%97%B6%E6%9C%BA/image-20200424133432059.png)

​        主要就是`handlerAdded` 和 `handlerRemoved`方法，这两个方法在将`handler`添加到`pipline`和从`pipline`上移除时调用，与数据传输无关，所以放在了这个接口里面。



## ChannelOutboundHandler：

![image-20200424133853407](/img/in/2020-04-24-%E8%81%8A%E4%B8%80%E8%81%8AChannelHandler%E9%87%8C%E9%9D%A2%E6%96%B9%E6%B3%95%E8%A2%AB%E8%B0%83%E7%94%A8%E6%97%B6%E6%9C%BA/image-20200424133853407.png)

## ChannelInboundHandler：

![image-20200424134116481](/img/in/2020-04-24-%E8%81%8A%E4%B8%80%E8%81%8AChannelHandler%E9%87%8C%E9%9D%A2%E6%96%B9%E6%B3%95%E8%A2%AB%E8%B0%83%E7%94%A8%E6%97%B6%E6%9C%BA/image-20200424134116481.png)



# 调用逐个分析

## handlerAdded

​		查看`DefaultChannelPipeline`类的`addLast()`方法。这个方法会先将`handler`封装成`HandlerContext`，然后添加到`pipline`尾部，最后调用`callHandlerAdded0`方法回掉`handler`的`handlerAdded`方法。

​		当发现此handler还没有registered，则将上面的操作创建成任务，等到registered后再执行。

```java
 @Override
    public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler);
			
            newCtx = newContext(group, filterName(name, handler), handler);

            addLast0(newCtx);

            // If the registered is false it means that the channel was not registered on an eventLoop yet.
            // In this case we add the context to the pipeline and add a task that will call
            // ChannelHandler.handlerAdded(...) once the channel is registered.
            if (!registered) {
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                callHandlerAddedInEventLoop(newCtx, executor);
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    }
```



## handlerRemove

​		查看`DefaultChannelPipeline`的`remove`方法，会先从`pipline`链上找到`handler`对应的`context`，然后移除`context`，最后在方法`callHandlerRemoved0()`里回掉`handlerRemove()`。

```java
   private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
        assert ctx != head && ctx != tail;

        synchronized (this) {
            atomicRemoveFromHandlerList(ctx);

            // If the registered is false it means that the channel was not registered on an eventloop yet.
            // In this case we remove the context from the pipeline and add a task that will call
            // ChannelHandler.handlerRemoved(...) once the channel is registered.
            if (!registered) {
                callHandlerCallbackLater(ctx, false);
                return ctx;
            }

            EventExecutor executor = ctx.executor();
            if (!executor.inEventLoop()) {
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callHandlerRemoved0(ctx);
                    }
                });
                return ctx;
            }
        }
        callHandlerRemoved0(ctx);
        return ctx;
    }
```



## channelRegistered

查看`AbstractChannel`内部类`unsafe`的下面方法,为了看起来更清晰我将它的注释删除了，此方法会先调用`doRegister()`方法，这个方法是要求子类实现的，然后在调用`pipeline.fireChannelRegistered()`来回掉pipline链上的`channelRegistered()`。

```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        //真实注册子类实现此方法
        doRegister();
        neverRegistered = false;
        registered = true;
        //没register之前添加到pipline上的handler，在这里真实添加
        pipeline.invokeHandlerAddedIfNeeded();
        safeSetSuccess(promise);
        //回掉channelRegistered方法
        pipeline.fireChannelRegistered();
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

​	

`pipeline.fireChannelRegistered()`方法会首先调用`head`的`channelRegister()`,然后head会传递给下一个`inChannelHandler`节点。

```java
@Override
public final ChannelPipeline fireChannelRegistered() {
    AbstractChannelHandlerContext.invokeChannelRegistered(head);
    return this;
}
```





## channelActive

​		此方法在`channel`处理连接成功后回掉，如在程序内连接到其他网站，连接成功后回掉此方法。

因为之有`客户端channel`才能连接到其他服务器，`serverChannel`是被动连接的，所以此方法对`serverChanel`是无效的。

​		查看`AbstractNioChannel`内部类`unsafe`的下面方法。

```java
@Override
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    try {
        if (connectPromise != null) {
            // Already a connect in process.
            throw new ConnectionPendingException();
        }

        boolean wasActive = isActive();
        //进行连接，
        if (doConnect(remoteAddress, localAddress)) {
            fulfillConnectPromise(promise, wasActive);
        } else {
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;

            // Schedule connect timeout.
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
                connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                    @Override
                    public void run() {
                        ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                        ConnectTimeoutException cause =
                                new ConnectTimeoutException("connection timed out: " + remoteAddress);
                        if (connectPromise != null && connectPromise.tryFailure(cause)) {
                            close(voidPromise());
                        }
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }

            promise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isCancelled()) {
                        if (connectTimeoutFuture != null) {
                            connectTimeoutFuture.cancel(false);
                        }
                        connectPromise = null;
                        close(voidPromise());
                    }
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```



​		此方法会调用子类的`doConnect()`方法，如果连接成功了`doConnect`返回true，如果还没连接成功，`doConnect`会注册`OP_CONNECT`事件到select里并返回false。

​		如果连接成功了，此方法会调用`fulfillConnectPromise()`，这里面回掉`pipline的channelActive()`，如果没连接成功，则添加超时时间的定时任务，超时事件内还没成功就取消连接并报错，并且没连接成功会注册`OP_CONNECT`事件到select内，当程序发现连接成功时，会处理`CONNECT`事件处理方法在下面。

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            return;
        }
        if (eventLoop == this) {
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();

        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            //在这里处理OP_CONNECT事件，调用finishConnect方法
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        if ((readyOps & SelectionKey.OP_WRITE) != 0) {

            ch.unsafe().forceFlush();
        }

        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```



​		遇到`op_connect`事件就是调用`unsafe.finishConnect()`方法，在这个方法里调用`fulfillConnectPromise`方法里回掉`channelActive`方法。

```java
@Override
public final void finishConnect() {
    // Note this method is invoked by the event loop only if the connection attempt was
    // neither cancelled nor timed out.

    assert eventLoop().inEventLoop();

    try {
        boolean wasActive = isActive();
        doFinishConnect();
        fulfillConnectPromise(connectPromise, wasActive);
    } catch (Throwable t) {
        fulfillConnectPromise(connectPromise, annotateConnectException(t, requestedRemoteAddress));
    } finally {
        // Check for null as the connectTimeoutFuture is only created if a connectTimeoutMillis > 0 is used
        // See https://github.com/netty/netty/issues/1770
        if (connectTimeoutFuture != null) {
            connectTimeoutFuture.cancel(false);
        }
        connectPromise = null;
    }
}
```





## read

上面调用了`pipline.channelActive`里，首先找到的`head`,它的方法里有一个`readIfIsAutoRead()`,如果此`channel`设置的是自动读，他会从`pipline`的尾部`tail`开始调用`read`方法，如果我们没重写`read`方法，最终会调用到`head`的`read`方法。

```java
        @Override
        public void channelActive(ChannelHandlerContext ctx) {
            ctx.fireChannelActive();

            readIfIsAutoRead();
        }

      private void readIfIsAutoRead() {
            if (channel.config().isAutoRead()) {
                channel.read();
            }
        }

           @Override
        public final ChannelPipeline read() {
            tail.read();
            return this;
        }
```



最终调用到下面代码，注册`readInterestOp`事件到`select`内。

```java
        @Override
        public final void beginRead() {
            assertEventLoop();

            if (!isActive()) {
                return;
            }

            try {
                doBeginRead();
            } catch (final Exception e) {
                invokeLater(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.fireExceptionCaught(e);
                    }
                });
                close(voidPromise());
            }
        }

        @Override
        protected void doBeginRead() throws Exception {
            // Channel.read() or ChannelHandlerContext.read() was called
            final SelectionKey selectionKey = this.selectionKey;
            if (!selectionKey.isValid()) {
                return;
            }

            readPending = true;

            final int interestOps = selectionKey.interestOps();
            if ((interestOps & readInterestOp) == 0) {
                selectionKey.interestOps(interestOps | readInterestOp);
            }
        }
```



​		`doBeginRead`方法使用的`readInterestOp`并不一定是`op_read`，别被它的名称迷惑了，对于`客户端的channel`他是`op_read`，对于`serverChannel`他就是`op_accept`,是通过构造方法传进来的。

​		这样对于服务器在`bind`成功后就可以`accept`了，对于`client`在`connect`成功后就可以`read`了。





## channelRead 和 channelReadComplate

​		这两个要在一起将，在客户端模式下，调用read注册`op_read`事件，下面是当有内容需要读取时的逻辑

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

 // 这里可以看出当是OP_READ 或者 OP_ACCEPT时均回掉unsafe.read();
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```



​		这里可以看出当是`OP_READ` 或者 `OP_ACCEPT`时均回掉`unsafe.read()`，虽然调用的方法名称是`read()`，但对于客户端来说是`read`数据，对于服务器来说是`accept`，处理逻辑是写在`unsafe`子类里面的，

下面先看客户端的

```java
 @Override
        public final void read() {
            final ChannelConfig config = config();
            if (shouldBreakReadReady(config)) {
                clearReadPending();
                return;
            }
            final ChannelPipeline pipeline = pipeline();
            final ByteBufAllocator allocator = config.getAllocator();
            final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
            allocHandle.reset(config);

            ByteBuf byteBuf = null;
            boolean close = false;
            try {
                do {
                    byteBuf = allocHandle.allocate(allocator);
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    if (allocHandle.lastBytesRead() <= 0) {
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        if (close) {
                            // There is nothing left to read as we received an EOF.
                            readPending = false;
                        }
                        break;
                    }

                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                } while (allocHandle.continueReading());

                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (close) {
                    closeOnRead(pipeline);
                }
            } catch (Throwable t) {
                handleReadException(pipeline, byteBuf, t, close, allocHandle);
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
    }
```



下面是服务端的

```java
 @Override
    public void read() {
        assert eventLoop().inEventLoop();
        final ChannelConfig config = config();
        final ChannelPipeline pipeline = pipeline();
        final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
        allocHandle.reset(config);

        boolean closed = false;
        Throwable exception = null;
        try {
            try {
                do {
                    int localRead = doReadMessages(readBuf);
                    if (localRead == 0) {
                        break;
                    }
                    if (localRead < 0) {
                        closed = true;
                        break;
                    }

                    allocHandle.incMessagesRead(localRead);
                } while (allocHandle.continueReading());
            } catch (Throwable t) {
                exception = t;
            }

            int size = readBuf.size();
            for (int i = 0; i < size; i ++) {
                readPending = false;
                pipeline.fireChannelRead(readBuf.get(i));
            }
            readBuf.clear();
            allocHandle.readComplete();
            pipeline.fireChannelReadComplete();

            if (exception != null) {
                closed = closeOnReadError(exception);

                pipeline.fireExceptionCaught(exception);
            }

            if (closed) {
                inputShutdown = true;
                if (isOpen()) {
                    close(voidPromise());
                }
            }
        } finally {
            // Check if there is a readPending which was not processed yet.
            // This could be for two reasons:
            // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
            // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
            //
            // See https://github.com/netty/netty/issues/2254
            if (!readPending && !config.isAutoRead()) {
                removeReadOp();
            }
        }
    }
}
```





​		从代码能看出对于`client`的`channel`多次读取数据调用`pipeline.fireChannelRead(byteBuf)`,最后调用一次`pipeline.fireChannelReadComplete()`，

​		对于server模式的channel，也有read方法，但它读取到的不是数据而是子连接，然后调用`pipline的fireChannelRead`方法处理子连接，具体处理逻辑在`ServerBootStrap`里面，就是对子连接进行config，然后注册到serverChannel相同的eventLoop里面。





## channelInactive，channelUnregistered，handlerRemoved

有很多因素会导致`channelInactive`，当连接断开，报错，主动断开等等，会依次回掉上面三个方法。

可以查看`pipline.close()`方法，逻辑不复杂，先关闭`java`的`channel`，并且从`select`中删除注册的`channel`。



## bind

​		这个方法是socket作为server注册时，绑定到本地端口时使用的，查看`AbstractBootstrap`的bind方法，会调用pipline的bind方法，这属于出站消息，先调用tail的bind方法，如果没重写此方法，最终调用head的bind方法。

```java
@Override
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();

    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    // See: https://github.com/netty/netty/issues/576
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
        // Warn a user about the fact that a non-root user can't receive a
        // broadcast packet on *nix if the socket is bound on non-wildcard address.
        logger.warn(
                "A non-root user can't receive a broadcast packet if the socket " +
                "is not bound to a wildcard address; binding to a non-wildcard " +
                "address (" + localAddress + ") anyway as requested.");
    }

    boolean wasActive = isActive();
    try {
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }

    safeSetSuccess(promise);
}
```



​	最终在`dobind`方法里，将`serverChannelbind`到指定端口上。





## connect

​		这个方法和bind方法类似，但只对于客户端的channel有效，使用`bootstrap`启动客户端连接时，最后一步会调用此方法，然后`bootstrap`就会创建`channel`，再注册到select里，再调用`pipline.connect()`方法来处理连接逻辑，如果不重写此方法，最终会在head里调用`unsafe.connect()`，实现连接。

​		**这里还涉及到dns解析的问题，因为connect之前要解析传入的域名为ip地址，代码在bootstrap的connect里面。**



## write

​		`write`也属于出站数据，我们一般会重写此方法来处理出站数据如加密，规范格式，自定义协议等。



# 总结

​		以上就是`netty4`版本里面的`channel`各个方法的调用过程，了解这些方法在什么时候会被调用对学习`netty`有很大帮助，在这之前需要先弄懂`pipline`里面的链式结构，还要了解`java`原生的`nio`的用法，才能看懂`netty`里面的用法。

