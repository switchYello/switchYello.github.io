---
layout:     post
title:      "guava eventBus使用和分析"
subtitle:   ""
date:       2019-08-09
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - java
---


## guava eventBus使用和分析

	eventBus就是类似于观察这模式，我们将监听器注册到eventBus内，调用eventBus的post（object）方法，
	将任何object对象发送到eventbus中，如果某个eventbus内的监听器对该object感兴趣，则会被回掉。并将该object作为参数传进去

## 为什么使用

	eventBus使用非常方便，效率高，且监听器只需加一个注解即可，不需要特定的方法名，不需要实现特定的接口，
	不需要知道自己身处于哪个eventbus中。

## 如何使用

```java

//任写一个类，任写一个方法，但只能有一个参数，且在方法上，加上Subscribe注解,这就是一个观察者
//参数类型为这个方法感兴趣的类型，注解标记此方法是一个监听者，一个类中可以多个方法都加此注解，这样每个方法都是一个监听者。

import com.google.common.eventbus.Subscribe;

/*这是一个监听者*/
public class Listerer3 {
	@Subscribe
	public void method1(Integer event) {
		System.out.println(event);
	}
}


//创建eventBus，注册监听者，发送数据
	public static void main(String[] args) {
		//创建eventBus
		EventBus e = new EventBus();
		//注册观察者
		e.register(new Listerer3());
		//发送消息，发送的消息是int类型的，这样监听其他类型的监听者就不会收到这条消息
		for (int i = 0; i < 3; i++) {
			e.post(i);
		}
	}

```

> ***注意 消息监听者不只是能够监听类型完全匹配的消息，如果监听的是父类型，而消息是子类型，也是可以收到的 ***




## 消息是根据什么找到观察者的

	消息可以使任意类型的，他根据观察者的参数类型进行匹配，如果观察者的能够处理则调用观察者的方法。否则不调用。
	其中如果观察者需要父类型的事件，而事件本身是子类型的同样可以触发。
	匹配方式是 `obj instanceof param` 而不是 `obj.getClass() == param.getClass()`
  


## eventbus底层是如何实现的

### 注册观察者

```
  void register(Object listener) {

	//通过反射获取该class及父类的的所有方法，检索带有 `@Subscribe`注解的方法
	//键为Subscriber注解方法的参数类型，值为同为此类型的所有method，包装成Subscriber对象
    Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);

	//循环遍历，并放入subscribers对象的Map<class,Set<>>中
    for (Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
      Class<?> eventType = entry.getKey();
      Collection<Subscriber> eventMethodsInListener = entry.getValue();

      CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);

      if (eventSubscribers == null) {
        CopyOnWriteArraySet<Subscriber> newSet = new CopyOnWriteArraySet<>();
        eventSubscribers =
            MoreObjects.firstNonNull(subscribers.putIfAbsent(eventType, newSet), newSet);
      }

      eventSubscribers.addAll(eventMethodsInListener);
    }
  }
```

### 取消注册观察者

```
  void unregister(Object listener) {

	//通过反射获取该class及父类的的所有方法，检索带有 `@Subscribe`注解的方法
	//键为Subscriber注解方法的参数类型，值为同为此类型的所有method，包装成Subscriber对象
    Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);

	//遍历map，从总的观察者集合中移除这些观察者，每个观察者等同于一个带注解的方法
    for (Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
      Class<?> eventType = entry.getKey();
      Collection<Subscriber> listenerMethodsForType = entry.getValue();

      CopyOnWriteArraySet<Subscriber> currentSubscribers = subscribers.get(eventType);
      if (currentSubscribers == null || !currentSubscribers.removeAll(listenerMethodsForType)) {
        throw new IllegalArgumentException(
            "missing event subscriber for an annotated method. Is " + listener + " registered?");
      }

      // don't try to remove the set if it's empty; that can't be done safely without a lock
      // anyway, if the set is empty it'll just be wrapping an array of length 0
    }
  }

```



### 发布消息

```
  public void post(Object event) {
	//根据消息类型，和消息的祖先类型，获取对应的观察者    
	Iterator<Subscriber> eventSubscribers = subscribers.getSubscribers(event);
    //至少有一个观察者，则循环回掉他们的方法，否则，发送一个DeadEvent，可以写一个观察者观察DeadEvent类，这样如果有某种类型的事件没有观察者可以检测到
	if (eventSubscribers.hasNext()) {
      dispatcher.dispatch(event, eventSubscribers);
    } else if (!(event instanceof DeadEvent)) {
      // the event had no subscribers and was not itself a DeadEvent
      post(new DeadEvent(this, event));
    }
  }

```


### 总结

	注册时会将对象的父类，父父类等的带注解的方法统统注册进来，得到一个参数类型为键，实际method为值的map ，即 map<class,Set<Method>>

	取消注册时，也会遍历祖先，因为有缓存了可能就不用遍历了，得到 map<class,List<Method>> ， 从总map中减去该map

	发布消息时，通过消息的类型，找到所有观察者，回掉他们的方法。不仅仅只是订阅了该消息类型的观察者，订阅了他父类的观察者同样可以收到消息。

### 注意

	两个相同的观察者不能注册多次，判断相同的方法是 方法所在的对象是同一个，方法相同。
	也就是说，一个类的同一个对象不能注册多次，同一个类可以new多个对象，每个new出来的新对象可以共同存在。