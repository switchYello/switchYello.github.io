---

layout:     post
title:      "Spring并发发送Websocket消息"
subtitle:   ""
date:       2020-06-07
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- spring
- websocket
---

 

# Spring并发发送Websocket消息



​		服务器的`Websocket`被客户端连接后，会创建一个`WebsocketSession`表示客户端连接，如想向客户端发送消息直接使用`WebsocketSession`发送即可。但是按照协议规范这个类是不是线程安全的，且发送过程是堵塞式的。

在此`Spring`提供一个代理类，他能处理线程安全问题，

他就是`org.springframework.web.socket.handler.ConcurrentWebSocketSessionDecorator`。



## 构造方法

`delegate` 需要代理的session

`sendTimeLimit`  表示发送单个消息的最大时间

`bufferSizeLimit`  表示发送消息的队列最大字节数，不是消息的数量而是消息的总大小

`overflowStrategy`  表示大小超过限制时的策略，默认是断开连接，还有个选项就是丢弃最老的数据，直到大小满足

```java
	public ConcurrentWebSocketSessionDecorator(
			WebSocketSession delegate, int sendTimeLimit, int bufferSizeLimit, OverflowStrategy overflowStrategy) {

		super(delegate);
		this.sendTimeLimit = sendTimeLimit;
		this.bufferSizeLimit = bufferSizeLimit;
		this.overflowStrategy = overflowStrategy;
	}	
```





## 发送方法

​		这个方法允许多线程调用，每个线程调用此方法时会对两个标志位`limitExceeded` ， `closeInProgress`进行检测，第一个表示已超过最大限制（时间或总大小），第二个表示此session正在关闭中。

​		然后将数据放入队列里，调用`tryFlushMessageBuffer`发送队列里的数据。当然只有第一个到达的线程来发送数据，后面的线程因为获取不到锁，会对线队列大小和发送时间进行检测。

​		第一个获取到发送锁的线程，每次从队列里拿到一个消息，每次发送完检测标志位，因为发送过程是堵塞的，在发送期间，如果其他线程发现超时了，并不能直接打断它，而是设置`limitExceeded` 标志位为true，等待发送线程自己判断，并停止发送。



```java
@Override
public void sendMessage(WebSocketMessage<?> message) throws IOException {
	//检测标志位
    if (shouldNotSend()) {
		return;
	}
	//放入队列，增加大小
	this.buffer.add(message);
	this.bufferSize.addAndGet(message.getPayloadLength());

	do {
        //尝试发送消息，因为要加锁，只有一个线程负责发送所有消息
		if (!tryFlushMessageBuffer()) {
			if (logger.isTraceEnabled()) {
				logger.trace(String.format("Another send already in progress: " +
						"session id '%s':, \"in-progress\" send time %d (ms), buffer size %d bytes",
						getId(), getTimeSinceSendStarted(), getBufferSize()));
			}
            //没获取锁的线程，对当前buffer和时间检测
			checkSessionLimits();
			break;
		}
	}
	while (!this.buffer.isEmpty() && !shouldNotSend());
}

private boolean shouldNotSend() {
    return (this.limitExceeded || this.closeInProgress);
}


private boolean tryFlushMessageBuffer() throws IOException {
    
   	if (this.flushLock.tryLock()) {
        try {
            //死循环发送消息
            while (true) {
                WebSocketMessage<?> message = this.buffer.poll();
                //每次发送消息前，先检测是否允许发送
                if (message == null || shouldNotSend()) {
                    break;
                }
                this.bufferSize.addAndGet(-message.getPayloadLength());
                this.sendStartTime = System.currentTimeMillis();
                getDelegate().sendMessage(message);
                this.sendStartTime = 0;
            }
        }
        finally {
            this.sendStartTime = 0;
            this.flushLock.unlock();
        }
        return true;
    }
    return false;
}

//检查是否超时，是否超过队列限制的部分
private void checkSessionLimits() {
    if (!shouldNotSend() && this.closeLock.tryLock()) {
        try {
            //检测是否发送超时
            if (getTimeSinceSendStarted() > getSendTimeLimit()) {
                String format = "Send time %d (ms) for session '%s' exceeded the allowed limit %d";
                String reason = String.format(format, getTimeSinceSendStarted(), getId(), getSendTimeLimit());
                limitExceeded(reason);
            }
            //检测buffer大小，根据策略要么抛异常，要么丢弃数据
            else if (getBufferSize() > getBufferSizeLimit()) {
                switch (this.overflowStrategy) {
                    case TERMINATE:
                        String format = "Buffer size %d bytes for session '%s' exceeds the allowed limit %d";
                        String reason = String.format(format, getBufferSize(), getId(), getBufferSizeLimit());
                        limitExceeded(reason);
                        break;
                    case DROP:
                        int i = 0;
                        while (getBufferSize() > getBufferSizeLimit()) {
                            WebSocketMessage<?> message = this.buffer.poll();
                            if (message == null) {
                                break;
                            }
                            this.bufferSize.addAndGet(-message.getPayloadLength());
                            i++;
                        }
                        if (logger.isDebugEnabled()) {
                            logger.debug("Dropped " + i + " messages, buffer size: " + getBufferSize());
                        }
                        break;
                    default:
                        // Should never happen..
                        throw new IllegalStateException("Unexpected OverflowStrategy: " + this.overflowStrategy);
                }
            }
        }
        finally {
            this.closeLock.unlock();
        }
    }
}

//超过限制则抛异常
private void limitExceeded(String reason) {
    this.limitExceeded = true;
    throw new SessionLimitExceededException(reason,CloseStatus.SESSION_NOT_RELIABLE);
}

```


​		上面代码能看到，一旦有一条消息发送超时，或者发送数据大于限制，`limitExceeded` 标志位就会被设置成true，标志这这个`session`被关闭，后面的发送调用都是直接返回不处理，但只是被标记为关闭连接本身可能实际上并没有关闭，这是一个坑点需要注意。可以在发送的代码里捕捉`SessionLimitExceededException`并主动关闭连接。