---
layout:     post
title:      "Spock测试框架中的mock类型"
subtitle:   ""
date:       2022-08-19
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- Spring
- Spock
- 单元测试
typora-root-url: ..
---


# Spock测试框架中的mock类型



​	最近使用Spock做单元测试比较多，但是对其中的Mock理解的比较混乱，今天抽些时间对其中的测试类型做测试，以此来揭示其中的区别。



##	认识mocking 和 stubing 和 Spying

​	mocking翻译成中文就是 "模拟"，stubing翻译成中文就是"存根"，spying翻译成中文就是"间谍"。



- 模拟拥有的能力是：对方法调用进行验证，在spock中表现形式如下。

```groovy
    1 * subscriber.receive("hello")

    1 * subscriber.receive("hello")
    |   |          |       |
        |   |          |       目标参数
    |   |          目标方法
    |   目标 Mock 对象
    调用次数
```

​	这就是'模拟'的能力，它的关注点是方法调用次数，调用参数等等的判断，这里不做赘述。



- 存根拥有的能力是：控制对象的返回值，在spock中的表现形式如下。

```groovy
subscriber.receive("message1") >> "ok"
subscriber.receive("message2") >> "fail"
```

​	这就是存根的能力，spock使用 `>>`作为特征，后面可以返回固定值，或者闭包，或者链式多个值。



- 间谍的能力是：工作在真实对象和调用方之间

间谍这个词比较形象，在编程里就是一层代理类，间谍代理真实对象，在调用链路中间做一些事情。



​	模拟+存根

同时使用模拟+存根也是可以的，在spock中是下面这种写法。这种写法下模拟和存根的效果是同时生效的。

```groovy
1 * subscriber.receive("message1") >> "ok"
```



## spock中的模拟和存根

​	spock提供三个方法用于生成mock对象，分别是Mock()，Stub()，Spay()。

**不要被他们的名字迷惑了，不要以为Mock对应上面的"模拟"能力，Stub对应"存根"能力，Spay对应"间谍"能力，这是不对的。**

​	

​	这三个方法都能返回一个对象，

其中Mock()生成的对象，它拥有上面所说的 "模拟"，“存根”两种能力，

Stub()生成的对象，只拥有上面说的"存根"一种能力

Spay()生成的对象具有"模拟"，“存根”，“间谍”三种能力



## spock配置类源码解读

​	请看这三种类型在源码中的枚举定义

```java
public enum MockNature {
  /**
   * A mock object whose method calls are verified, which instantiates class-based mock objects with Objenesis,
   * and whose strategy for responding to unexpected method calls is {@link ZeroOrNullResponse}.
   */
  MOCK(true, true, ZeroOrNullResponse.INSTANCE),

  /**
   * A mock object whose method calls are not verified, which instantiates class-based mock objects with Objenesis,
   * and whose strategy for responding to unexpected method calls is {@link EmptyOrDummyResponse}.
   */
  STUB(false, true, EmptyOrDummyResponse.INSTANCE),

  /**
   * A mock object whose method calls are verified, which instantiates class-based mock objects by calling a
   * real constructor, and whose strategy for responding to unexpected method calls is {@link CallRealMethodResponse}.
   */
  SPY(true, false, CallRealMethodResponse.INSTANCE);

  private final boolean verified;
  private final boolean useObjenesis;
  private final IDefaultResponse defaultResponse;

  MockNature(boolean verified, boolean useObjenesis, IDefaultResponse defaultResponse) {
    this.verified = verified;
    this.useObjenesis = useObjenesis;
    this.defaultResponse = defaultResponse;
  }
```



`MockNature`对象有三个参数，这三个参数的组合决定了这三种类型的特性。

`verified`：表示是否识别调用次数，调用参数验证。即是否能够使用 `1 * foo.doSome()` 这种样子的断言验证

`useObjenesis`：表示使用是否使用Objenesis工具创建mock对象。PS：有些类没有默认构造方法，所以通过反射无法创建出来，使用Objenesis工具能绕过构造方法直接创建对象。其中Spy是直接new的，其余两个使用Objenesis工具

`defaultResponse`：表示生成的mock对象的默认返回策略。Mock尽量返回0或null，Stub尽量返回对象的默认值，Spy透传给真实类。例如方法返回值是字符串类型的，Mock会返回null，Stub会返回空字符串，Spy会透传给真实对象。



## 做个表

| 方式 | 支持调用验证 | 支持方法mock | mock策略                          | 返回值策略                           |
| ---- | ------------ | ------------ | --------------------------------- | ------------------------------------ |
| Mock | Y            | Y            | 使用Objenesis忽略构造方法创建对象 | 尽量返回0或者null                    |
| Stub | N            | Y            | 使用Objenesis忽略构造方法创建对象 | 尽量返回默认值，如空字符串，空集合等 |
| Spy  | Y            | Y            | 使用构造方法new对象               | 委托给真实对象获取返回值             |



​	如上面所说，Stub的``verified``参数是false，所以它没有"模拟的能力"。Spy的``useObjenesis``是false，因为要使用构造方法创建被代理对象。

​	同时他们对待返回值处理策略也不相同，Mock尽量返回null值，Stub尽量返回默认值，Spay不自己处理返回值。



## 踩坑记录

### 	方法mock

​	单元测试时，我们要对某个方法进行mock，也就是上面说到的"存根"能力。

​	请看下面的测试案例，when块和then块中均对test方法进行了mock。按照正常人思维要么会报错，要么第一个生效。其实并不是这样。

​	 按照官方的说法，程序编译时会优先解析then块中的mock，所以then块中优先级更高，when块中的mock不会生效。

```groovy
@Slf4j
class TestSpockNoSpring extends Specification {

    SpockTestService spockTestService = Mock()

    @IgnoreRest
    def "test"() {
        when:
        spockTestService.test() >> 'mock'		//这里优先级比then块中的低

        def doTest = spockTestService.test()
        log.info('CallDoTest:{}', String.valueOf(doTest))

        then:
        spockTestService.test()  >> 'mock at then'	//这里的mock生效
    }

}
```



### 	隐式的mock

​	看下面这里案例，看起来应该是when中的"存根"能力生效了，then块中的"模拟"能力剩下了，但其实并不是这样。

​	还记得上面提到，spock支持"模拟"+"存根"同时使用吗，then块中虽然只使用了"模拟"能力，其实隐式的使用了"存根"，then块中的语句隐式转换成了这样 `1 * spockTestService.test() >> null`。结合上面提到的优先级，所以when块中的mock失效。

```groovy
@Slf4j
class TestSpockNoSpring extends Specification {

    SpockTestService spockTestService = Mock()

    @IgnoreRest
    def "test"() {

        when:
        spockTestService.test() >> 'mock'	//此mock失效

        def doTest = spockTestService.test()
        log.info('CallDoTest:{}', String.valueOf(doTest))

        then:
        1 * spockTestService.test()
    }
}
```