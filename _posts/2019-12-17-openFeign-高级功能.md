---


layout:     post
title:      "Feign高级功能"
subtitle:   "Feign NB"
date:       2019-12-17
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- openfeign
- Http客户端
---

# Feign高级功能

* TOC
{:toc}
## 动态修改请求地址

像这样，创建接口时放置一个类型为`java.net.URI`的参数，这样真正发送请求时就会以此uri为准。

```java
    interface dyPath {
        @RequestLine("GET /get/item")
        String getItem(URI uri);
    }
```



## 动态请求配置Options

放置一个类型为`Options`的参数在接口中，这样发送请求时就会使用此类作为配置，否则会使用默认的配置

```java
    interface dyPath {
        @RequestLine("GET /get/item")
        String getItem(Request.Options options);
    }
```



- 源码在这里:

```java
//feign.SynchronousMethodHandler#191  
Options findOptions(Object[] argv) {
    if (argv == null || argv.length == 0) {
      return this.options;
    }
    return Stream.of(argv)
        .filter(Options.class::isInstance)
        .map(Options.class::cast)
        .findFirst()
        .orElse(this.options);
  }
```



## @QueryMap注解

- 这个注解如果注在Map上，此Map的键只能是String类型的如`Map<String,Object>`，其他类型会报错。

    ```java
    //源码在这里: feign.Contract.BaseContract#checkMapKeys
        if (keyClass != null) {
            checkState(String.class.equals(keyClass),
                       "%s key must be a String: %s", name, keyClass.getSimpleName());
        }
    ```

    

- 如果在`@RequestLine`的链接中放置了参数， `@QueryMap`中有同名参数是一个空集合，则会把前面的参数删掉，所以想要删除某参数，也可以直接设置一个空集合参数，同理添加heads也有这样的删除策略。

    ```java
        interface Fq {
            @RequestLine(value = "GET /cms/materials/create?username={username}")
            String getItemCoupon(@Param(value = "username") String username
                            , @QueryMap Map<String, Object> params);
        }
    
    //调用过程
         HashMap<String, Object> params = new HashMap<>();
         params.put("username", Collections.emptyList());
         String itemCoupon = target.getItemCoupon("小明", params);
    
    //参数Map里面有一‘username’和链接上的参数同名了，且是一个空集合，即使链接上的`username`有值，也会导致username参数被删除
    
    //实际访问地址 GET http://localhost/create HTTP/1.1
    
    ```

- `@QueryMap`解析出来的参数只会拼接在url后面，不会当成请求体,无论请求类型是什么。

    >  有时候post请求不希望参数拼在链接后面，就不能用这个了。



## @Param注解注意事项

注解标记的参数，如果没有在任何地方被使用，则会被放入`MethodMetadata`的`formParams`中。

```java
      super.registerParameterAnnotation(Param.class, (paramAnnotation, data, paramIndex) -> {
        String name = paramAnnotation.value();
        checkState(emptyToNull(name) != null, "Param annotation was empty on param %s.",
            paramIndex);
        nameParam(data, name, paramIndex);
        Class<? extends Param.Expander> expander = paramAnnotation.expander();
        if (expander != Param.ToStringExpander.class) {
          data.indexToExpanderClass().put(paramIndex, expander);
        }
        data.indexToEncoded().put(paramIndex, paramAnnotation.encoded());
        //这里检测，没有任何地方使用它，则放入formParams中
        if (!data.template().hasRequestVariable(name)) {
          data.formParams().add(name);
        }
      });
```



如下面写法的两个参数`name`和`age` 就是未使用因为`RequestLine`和`Headers`没有用它:

```java
    interface TestJson {
        @RequestLine(value = "POST /test")
        @Headers("Content-Type:application/json")
        Response getItemCoupon(@Param("name") String name,@Param("age") Integer age);
    }
```



在发送请求时，会将他们组成一个map然后传入encoder中进行编码，编码后的结果作为请求的`body`使用，而默认的encoder不能处理map类型的，会报错.

`feign.codec.EncodeException: class java.util.LinkedHashMap is not a type supported by this encoder`

添加一个`JacksonEncoder`后可以将他们序列化成JSON字符串作为请求`body`,即使是GET请求也会变成请求body，所以如果这不是你想要的结果，就不要添加了参数却不使用它。

如果不想将他们转成JSON，可以自己写encoder来处理 。



## 没有加任何注解的参数

默认情况下允许存在`一个`没有注解的参数（注意URI不包括在内），将它的参数索引设置成`MeteDate`的`bodyIndex`，发送请求前，会把这个参数的值取出来，然后调用`encoder`进行编码，编码的结果当成请求`body`使用。

```java
// feign.Contract.BaseContract#parseAndValidateMetadata()

//参数类型为接口的class类，需要解析的method
protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
      MethodMetadata data = new MethodMetadata();
      data.targetType(targetType);
      data.method(method);
      data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
      data.configKey(Feign.configKey(targetType, method));

      if (targetType.getInterfaces().length == 1) {
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
      }
    //解析类上的注解
      processAnnotationOnClass(data, targetType);
	//循环解析方法上的所有注解
      for (Annotation methodAnnotation : method.getAnnotations()) {
        processAnnotationOnMethod(data, methodAnnotation, method);
      }
    
      if (data.isIgnored()) {
        return data;
      }
     //获取方法参数类型
      Class<?>[] parameterTypes = method.getParameterTypes();
     //获取方法泛型类型
      Type[] genericParameterTypes = method.getGenericParameterTypes();
	 //获取方法的参数注解，是二维数组，因为每个参数可以有多个注解，没有注解则为空数组
      Annotation[][] parameterAnnotations = method.getParameterAnnotations();
      int count = parameterAnnotations.length;
      for (int i = 0; i < count; i++) {
        boolean isHttpAnnotation = false;
        //解析参数的注解
        if (parameterAnnotations[i] != null) {
          isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
        }

        if (isHttpAnnotation) {
          data.ignoreParamater(i);
        }
	
        if (parameterTypes[i] == URI.class) {
          data.urlIndex(i);
        } else if (!isHttpAnnotation && parameterTypes[i] != Request.Options.class
            //如果第i位参数没有被引用过，就将它设为bodyIndex
            //这里只有没注解的参数才能验证成功
              && !data.isAlreadyProcessed(i)) {
            //在设置bodyIndex前保证formParams是空的，也就是没有无引用的@Param标记的参数
          checkState(data.formParams().isEmpty(),
              "Body parameters cannot be used with form parameters.");
          checkState(data.bodyIndex() == null, "Method has too many Body parameters: %s", method);
          data.bodyIndex(i);
          data.bodyType(Types.resolve(targetType, targetType, genericParameterTypes[i]));
        }
      }

      if (data.headerMapIndex() != null) {
        checkMapString("HeaderMap", parameterTypes[data.headerMapIndex()],
            genericParameterTypes[data.headerMapIndex()]);
      }

      if (data.queryMapIndex() != null) {
        if (Map.class.isAssignableFrom(parameterTypes[data.queryMapIndex()])) {
          checkMapKeys("QueryMap", genericParameterTypes[data.queryMapIndex()]);
        }
      }

      return data;
    }

```



默认的`encoder`只能处理String和byte[]类型的参数，其他参数需要自定义`encoder`来处理。

如下面两种情况，存在没注解的参数，会把他们转成请求体使用：

```java
    interface TestJson {
        @RequestLine(value = "POST /test")
        Response getItemCoupon(String name);
    }
```

```java
    interface TestJson {
        @RequestLine(value = "POST /test")
        Response getItemCoupon(Map<String,Object> name);
    }
```



> 需要注意的一点是，这种方式构建请求`body`和上面[@Param注解注意事项](#param注解注意事项)里面未使用的`@Param`标记构建请求`body`的方式一起存在的话，有两种可能。
>
> 如果`@Param`的参数在无注解参数的前面，则按照上面代码49行会抛出`Body parameters cannot be used with form parameters`,提示body参数不能和表单参数一起用。
>
> 如果无注解参数放在`@Param`参数前面，无注解方式会被忽略，发送请求体由`@Param`的参数决定。	



## 三种设置请求体的优先级

一共三种设置请求体的方式，分别是：

- 1.使用`@Body`注解
- 2.使用没有任何注解的参数 （这样的参数只能存在一个）
- 3.使用没有被使用的`@Param`标记的参数 （这个可以有多个参数）



他们之间的关系很复杂不能简单的用优先级来排序了。

```java
      super.registerMethodAnnotation(Body.class, (ann, data) -> {
        String body = ann.value();
        if (body.indexOf('{') == -1) {
          data.template().body(body);
        } else {
          data.template().bodyTemplate(body);
        }
      });
```

> 可以看到，如果有body注解，如果body注解的值是固定的，则将值设置为body，如果是可变的设置为bodyTemplate



> 根据上面[Param注解注意事项](#Param注解注意事项)源码看出，如果`@Param`标记的参数没被使用，则放入`data.formParams()`中



> 在根据上面讲的[没有加任何注解的参数](#没有加任何注解的参数), 则会将他设置为`data.bodyIndex(i)`



且bodyIndex只能设置一个，设置bodyIndex时会断言formParams为空，所以后两个同时使用的话，要保证后者放在参数的前面，先解析才不会报错。

调用时是这样的:`feign.ReflectiveFeign.ParseHandlersByName#apply`

```java
//如果formarams中有值，bodyTemplate没值，则是哟个formarams构建请求体
if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
          buildTemplate =
              new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder, target);
//否则尝试是哟个bodyIndex构建请求体    
        } else if (md.bodyIndex() != null) {
          buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder, target);
//否则使用默认bodyTemplate构建请求体
        } else {
          buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder, target);
        }
```



> 所以他们的优先级是 
>
> formParams > bodyIndex
>
> bodyIndex优先级 > bodyTemplate
>
> bodyTemplate > formParams

简直是个三角形的关系。

但我觉得将无注解参数放在`@Param`参数前就不报错，应该数据bug。







