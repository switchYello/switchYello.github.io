---
layout:     post
title:      "project reactor中的onErrorXxx操作符"
subtitle:   ""
date:       2022-03-17
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- reactor
- onErrorMap
- onErrorResume
- onErrorReturn
- onErrorContinue
- onErrorStop
typora-root-url: ..
---


# project reactor中的onErrorXxx操作符



​	reactor项目中存在很多操作符其中一些与处理异常有关的操作符，它们都以onError开头，下面我们逐个分析他们。



- onErrorMap
- onErrorReturn
- onErrorResume



## 这三个是用来处理上游传递的错误信号

这三个是一组的，他们原理类似，分别是：

onErrorMap 需要传递一个 `Function<? super Throwable, ? extends Throwable> mapper`，他表示将upStream传递的错误转换成另一种错误返回，即替换upStream中产生的错误。要求Function函数返回值必须非空，否则报空指针。



onErrorReturn需要传递一个值，即fallback。它会将upStream传递的error信号吞掉，用给定的值替换向下传递，并且upStream已经发出error信号，也就是已经终止了。对于下游订阅者来说，感知不到错误信号，且此时fallback值就是是最后一个元素了。



onErrorResume 使用一个fallback 流代替upStream，即对于下游订阅者来说，原始流发出error信号后会被吞掉，然后接着从fallback流中订阅。即fallback流连接在upStream发生错误的位置继续产生元素。



​	这三个都是下游对错误信号的处理操作符，当接收到错误信号时如何向下游传递。默认情况下没有这三个操作符时，错误信号是透传给下游的，这三个操作符提供在中间拦截的能力。其中onErrorResume可以实现onErrorMap和onErrorReturn的功能。

​	这三个的实现原理如下：

```java
//当上游传递错误信号时		
@Override
public void onError(Throwable t) {
    if (!second) {
        second = true;

        Publisher<? extends T> p;

        //使用给定的Function创建一个新的生产者
        try {
            p = Objects.requireNonNull(nextFactory.apply(t),
                                       "The nextFactory returned a null Publisher");
        }
        catch (Throwable e) {
            Throwable _e = Operators.onOperatorError(e, actual.currentContext());
            _e = Exceptions.addSuppressed(_e, t);
            actual.onError(_e);
            return;
        }
        //继续订阅新的生产者。对下游来说没有感知到发生了错误信号
        p.subscribe(this);
    }
    else {
        actual.onError(t);
    }
}
```





- onErrorContinue

- onErrorStop

  

## 这两个是用来指示上游如何处理发生的错误的操作符

​		这两个是一组，他们与上面三个原理不同。他们会给Context设置上错误处理策略，当upStream产生错误准备调用下游的onError时，可以从context中获取该策略对错误进行处理。如设置的策略是onErrorContinue，则可以跳过此错误当作此元素没发生过一样继续处理下一个元素。如设置的策略是OnErrorStop则会将错误向下游传递，并取消上游的订阅。

​		虽然我们设置了错误处理策略，但只是对upStream的建议，upStream会不会使用它，downStream无法决定，只有一部分操作符会使用Context内设置的错误处理策略。



这两个操作符的原理如下:

​	调用`onErrorContinue`时，将OnNextFailureStrategy 设置到Context中

```java
//方法调用时，将OnNextFailureStrategy 设置到Context中
public final Mono<T> onErrorContinue(BiConsumer<Throwable, Object> errorConsumer) {
    BiConsumer<Throwable, Object> genericConsumer = errorConsumer;
    return subscriberContext(Context.of(
        OnNextFailureStrategy.KEY_ON_NEXT_ERROR_STRATEGY,
        OnNextFailureStrategy.resume(genericConsumer)
    ));
}
```



​	在某个操作符内部发生错误，如map操作符内调用onNext发生错误。

此时会使用`Operators.onNextError`处理该错误，如果处理过后返回值为null，则忽略错误向upStream请求下一个元素，如果返回值非null，则将错误传递下去。

```java
@Override
public void onNext(T t) {
	if (done) {
		Operators.onNextDropped(t, actual.currentContext());
		return;
	}
	R v;
	try {
		v = Objects.requireNonNull(mapper.apply(t),
				"The mapper returned a null value.");
	}
	catch (Throwable e) {
        //发生错误，使用此方法处理
		Throwable e_ = Operators.onNextError(t, e, actual.currentContext(), s);
		//如果返回值有值，将错误向下传递。如果返回null，则忽略错误继续从上游生产者中订阅
        if (e_ != null) {
			onError(e_);
		}
		else {
			s.request(1);
		}
		return;
	}
	actual.onNext(v);
}
```



​	Operators.onNextError 这里会先获取错误处理策略，根据策略做处理。

```java
@Nullable
public static <T> Throwable onNextError(@Nullable T value, Throwable error, Context context,
		Subscription subscriptionForCancel) {
	error = unwrapOnNextError(error);
	OnNextFailureStrategy strategy = onNextErrorStrategy(context);
	if (strategy.test(error, value)) {
		//some strategies could still return an exception, eg. if the consumer throws
		Throwable t = strategy.process(error, value, context);
		if (t != null) {
			subscriptionForCancel.cancel();
		}
		return t;
	}
	else {
		//falls back to operator errors
		return onOperatorError(subscriptionForCancel, error, value, context);
	}
}
```



​	默认我们不配置任何策略时，处理策略是 `OnNextFailureStrategy.STOP`，即取消订阅upStream，并将错误向下传递。

```java
static final OnNextFailureStrategy onNextErrorStrategy(Context context) {
	OnNextFailureStrategy strategy = null;
	BiFunction<? super Throwable, Object, ? extends Throwable> fn = context.getOrDefault(
			OnNextFailureStrategy.KEY_ON_NEXT_ERROR_STRATEGY, null);
	if (fn instanceof OnNextFailureStrategy) {
		strategy = (OnNextFailureStrategy) fn;
	} else if (fn != null) {
		strategy = new OnNextFailureStrategy.LambdaOnNextErrorStrategy(fn);
	}
	if (strategy == null) strategy = Hooks.onNextErrorHook;
	if (strategy == null) strategy = OnNextFailureStrategy.STOP;
	return strategy;
}
```





## 总结

onErrorMap

onErrorResume

onErrorReturn

onErrorContinue

onErrorStop

​		这五个操作符虽然都是以onError开头，但根据原理可以分为两类。一类是如何处理上游传递的error信号，但无论怎么处理，error信号产生后upStream就算完结了。另一类将错误处理策略设置在Context内，上游产生错误后可以根据策略决定是否终止生产。

​		并且onErrorMap，onErrorResume，onErrorReturn只对上游产生的错误负责，该操作符下游产生的错误无法被处理。

​		而onErrorContinue，onErrorStop操作符是在被订阅时将处理策略放置在Context内，而Context是同一个对象，也就说订阅过程中操作符下游发生错误，此时还没将策略放置到Context内，此操作符不生效。而订阅过程中上游发生错误，会以后处理数据时发生错误，因为该操作符被订阅时已经将处理策略放置到Context内了，所以策略总是生效的。

​		所以这两个操作符建议放在比较靠后的位置。

