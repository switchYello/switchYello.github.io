---
layout:     post
title:      "netty 不要要在future监听器的回掉函数中抛出异常"
subtitle:   ""
date:       2019-11-07
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - netty
---


### netty 不要要在future监听器的回掉函数中抛出异常

	promise身上添加的监听器，当promise完成过后，会回掉`notifyListeners();`方法。通知所有的监听器
	在监听器内抛出的异常都会被吃掉打印出来。

```

	//判断是否要通知监听器
   private void notifyListeners() {
        EventExecutor executor = executor();
        if (executor.inEventLoop()) {
            final InternalThreadLocalMap threadLocals = InternalThreadLocalMap.get();
            final int stackDepth = threadLocals.futureListenerStackDepth();
            if (stackDepth < MAX_LISTENER_STACK_DEPTH) {
                threadLocals.setFutureListenerStackDepth(stackDepth + 1);
                try {
                    notifyListenersNow();
                } finally {
                    threadLocals.setFutureListenerStackDepth(stackDepth);
                }
                return;
            }
        }

        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                notifyListenersNow();
            }
        });
    }

	//根据监听器类不同的处理方法
private void notifyListenersNow() {
    Object listeners;
    synchronized (this) {
        // Only proceed if there are listeners to notify and we are not already notifying listeners.
        if (notifyingListeners || this.listeners == null) {
            return;
        }
        notifyingListeners = true;
        listeners = this.listeners;
        this.listeners = null;
    }
    for (;;) {
        if (listeners instanceof DefaultFutureListeners) {
            notifyListeners0((DefaultFutureListeners) listeners);
        } else {
            notifyListener0(this, (GenericFutureListener<?>) listeners);
        }
        synchronized (this) {
            if (this.listeners == null) {
                // Nothing can throw from within this method, so setting notifyingListeners back to false does not
                // need to be in a finally block.
                notifyingListeners = false;
                return;
            }
            listeners = this.listeners;
            this.listeners = null;
        }
    }
}


	//循环遍历所有的监听器
private void notifyListeners0(DefaultFutureListeners listeners) {
    GenericFutureListener<?>[] a = listeners.listeners();
    int size = listeners.size();
    for (int i = 0; i < size; i ++) {
        notifyListener0(this, a[i]);
    }
}
//回掉operationComplete方法，捕获所有的Throwable并warn输出出来
  private static void notifyListener0(Future future, GenericFutureListener l) {
    try {
        l.operationComplete(future);
    } catch (Throwable t) {
        if (logger.isWarnEnabled()) {
            logger.warn("An exception was thrown by " + l.getClass().getName() + ".operationComplete()", t);
        }
    }
}

```

### 总结
	不要在监听器中抛出异常，那样不会生效果，但可以将异常传递出去