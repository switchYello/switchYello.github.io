---

layout:     post
title:      "Java的CopyOnWriteArrayList"
subtitle:   ""
date:       2020-06-06
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- java
- CopyOnWriteArrayList
---



# 

# Java的 CopyOnWriteArrayList



​	`CopyOnWriteArrayList`会在每次更新时更换底层数组，这样你使用增强for循环或者iterator遍历时，遍历的数组一定和你第一次获取的数组相同，不会出现遍历过程中修改`List`，导致`List`混乱的问题。



查看下`CopyOnWriteArrayList的forEach()`方法：

​	这个方法会获取底层数组作为局部变量保存，然后遍历此数组，并且不担心遍历期间数组被修改。

因为底层数组是只读的，每次修改都会创建新的代替。

```java
public void forEach(Consumer<? super E> action) {
    if (action == null) throw new NullPointerException();
    //获取底层数组，作为局部变量
    Object[] elements = getArray();
    int len = elements.length;
    for (int i = 0; i < len; ++i) {
        @SuppressWarnings("unchecked") E e = (E) elements[i];
        action.accept(e);
    }
}
```



查看下`Add`方法:

​	这是add方法，它创建一个长度等于原长度+1的数组，并将底层数组切换成新创建的数组。这样其他读线程遍历期间拿到的数组不会收到影响。

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        //创建新的数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        //将底层数组替换为这个数组，所以其他线程获取的旧数组不受影响
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

  final void setArray(Object[] a) {
        array = a;
    }
```





案例，遍历集合时需要操作集合：

​	这里存在一个User集合，send方法遍历集合调用`user.send`方法,但是send方法中有可能会从`list`中删除自身,这里用`CopyOnWriteArrayList`正合适，能保证所有元素都被遍历到，还能保证删除不会影响遍历。

```java
    List<User> list = new CopyOnWriteArrayList<>();

    public void send(String msg){
        for (User u : list) {
            try {
                u.send(msg);
            } catch (Exception e) {
                list.remove(u);
                send(e.getMessage());
            }
        }
    }

   class User {
       //此方法内有可能会将自己从list中删除
        void send(String msg) {
            if (new Random().nextBoolean()) {
                list.remove(this);
            }
        }
    }

```



还有种场景，就是处理`Websocket`连接时，遍历每个`WebsocketSession`发送消息，如果发送失败则关闭`session`,从集合中移除，然后广播关闭信息。就有可能在遍历集合途中修改集合，再遍历集合。



