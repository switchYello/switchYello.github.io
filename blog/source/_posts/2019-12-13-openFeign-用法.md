---


layout:     post
title:      "Feign用法教程"
subtitle:   "Feign文档翻译"
date:       2019-12-13
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- openfeign
- Http客户端
---

# Feign用法教程

* TOC
{:toc}



## Maven依赖

创建java项目，引入maven

```xml
        <!--核心包-->
		<dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-core</artifactId>
            <version>10.7.0</version>
        </dependency>
		<!--使用slf4j 打log-->
         <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-slf4j</artifactId>
            <version>10.7.0</version>
         <!--引入logback，版本自己选择-->
        </dependency>
              <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
        </dependency>
```



## 典型示例

```java
public class App {

    interface GitHub {
        @RequestLine("GET /repos/{owner}/{repo}/contributors")
        String contributors(@Param("owner") String owner, @Param("repo") String repo);
    }

    public static void main(String... args) {
        GitHub github = Feign.builder()
                .logger(new Slf4jLogger())
                .logLevel(Logger.Level.FULL)
                .target(GitHub.class, "https://api.github.com");
        
        String contributors = github.contributors("OpenFeign", "feign");
    }
    
}
```



## 支持的注解



| 注解         | target           | 说明                                                         |
| ------------ | ---------------- | ------------------------------------------------------------ |
| @RequestLine | 方法上           | 按照上面的示例一样，可以在此注解中表名请求method，和请求地址，并且可以使用`{表达式}`，这样的形式来从参数中提取数据，动态构造请求地址。 |
| @Param       | 参数上           | 和上面示例一样，标记一个参数，这样在各种表达式中才能使用     |
| @Headers     | 方法上<br />类上 | 这个注解有一个value参数，里面放请求头，参数中和`@requestLine`类似，可以使用表达式来动态创建，如果标记在方法上则此方法对应的请求有效，如果标记在类上，则整个类所有方法对应的请求有效。 |
| @QeuryMap    | 参数上           | 将一个map或pojo标记为查询参数，取提取数据扩展成查询参数拼接在url后面 |
| @HeaderMap   | 参数上           | 和queryyMap同理，将键值对当成head处理                        |
| @Body        | 方法上           | 用法和`RequestLine`类似，它的值会被当作请求体看待，如可以是一个json字符串，里面也可以用表达式 |



> ####  覆盖请求host
>
> ​		如果不希望`Feign.builder`这样生成接口代理类时指定请求的host，那么可以在方法上添加一个	`java.net.URI`的参数，此参数将覆盖builder生成代理类时传入的host。
>
> ```java
> @RequestLine("POST /repos/{owner}/{repo}/issues")
> void createIssue(URI host, Issue issue, @Param("owner") String owner, @Param("repo") String repo);
> ```



## 模板和表达式

​	Feign 表达式遵守 [这里](https://tools.ietf.org/html/rfc6570)定义的String表达式（Level 1），使用`Param`注解来扩展表达式

### 例子

```java
public interface GitHub {
  
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repository);
  
  class Contributor {
    String login;
    int contributions;
  }
}

public class MyApp {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                         .target(GitHub.class, "https://api.github.com");
    
    /* owner和 repository两个参数会用来扩展RequestLine中的表达式模板
     * 
     * 真实请求的链接就会变成
   	 *https://api.github.com/repos/OpenFeign/feign/contributors
     */
    github.contributors("OpenFeign", "feign");
  }
}
```

> 表达式必须用大括号`{}`包起来的形式，也可以使用正则来限制表达式解析后的值，正则使用冒号`:`进行分隔，比如 `{owner:[a-zA-Z]*}` 这样来限制`owner`必须是大小写字母。即前面的值必须满足后面的正则表达式， feign.template.Expressions#125



### 请求路径和参数扩展

`RequestLine` 和`QueryMap`模板符合 [URI Template -RFC6570](https://tools.ietf.org/html/rfc6570)规范，下面需要注重指出:

* 未被解析的表达式将被忽略

* 除非使用`@Param`注解的参数对参数标记为`encoded`，否则都要进行 pct-encoded。

    > 这里的意思是使用`@Param(value = "owner",encoded = true) String owner`这样的形式表名该参数已经被编码过了，没标记的会被自动编码。新版本encoded这个参数被弃用了，因为新版本能自动检测值有没有被编码，如果已经编码了就不会再编码，如果没编码就进行编码，所以不使用此参数，让其默认false即可



#### Undefined vs Empty Values

Undefined的意思是 `null`，根据 [URI Template - RFC 6570](https://tools.ietf.org/html/rfc6570)规范，可以为表达式提供一个空值。当Feign解析一个表达式时，它首先确认该值知否被定义，如果被定义了，则`query parameter`会被保留，如果没有被定义，则`query parameter`会被移除，下文会详细解释。

* 表达式出现在path中

    * 如果是空字符串

        ```java
         interface Fq {
                @RequestLine(value = "GET /item/{id}/create")
                String getItemCoupon(@Param("id") String id);
            }
        
            public static void main(String... args) {
                Fq target = Feign.builder()
                        .logger(new Slf4jLogger())
                        .logLevel(Logger.Level.FULL)
                        .target(Fq.class, "http://localhost");
                String itemCoupon = target.getItemCoupon("");
                
         //结果  GET http://localhost/item//create
        ```

    * 如果是null

        ```java
            interface Fq {
                @RequestLine(value = "GET /item/{id}/create")
                String getItemCoupon(@Param("id") String id);
            }
        
            public static void main(String... args) {
                Fq target = Feign.builder()
                        .logger(new Slf4jLogger())
                        .logLevel(Logger.Level.FULL)
                        .target(Fq.class, "http://localhost");
                String itemCoupon = target.getItemCoupon(null);
        //结果 GET http://localhost/item//create 
        ```

    * 如果是`+`或`/` 这种特殊字符

        ```java
             interface Fq {
                @RequestLine(value = "GET /item/{id}/create")
                String getItemCoupon(@Param("id") String id);
            }
        
            public static void main(String... args) {
                Fq target = Feign.builder()
                        .logger(new Slf4jLogger())
                        .logLevel(Logger.Level.FULL)
                        .target(Fq.class, "http://localhost");
                String itemCoupon = target.getItemCoupon("a+b/c");
         
         //结果  GET http://localhost/item/a+b/c/create HTTP/1.1
        ```

        

        > 由上我们可以看出，如果参数表达式出现在path中，是不会被转码的，且如果是null的会变成空字符串。

* 表达式出现在queryString中

    * 空字符串

        ```java
            interface Fq {
                @RequestLine(value = "GET /item?id={id}")
                String getItemCoupon(@Param("id") String id);
            }
        
            public static void main(String... args) {
                Fq target = Feign.builder()
                        .logger(new Slf4jLogger())
                        .logLevel(Logger.Level.FULL)
                        .target(Fq.class, "http://localhost");
                String itemCoupon = target.getItemCoupon("");
        //结果  GET http://localhost/item?id HTTP/1.1
        ```

    * 如果是null

        ```java
            interface Fq {
                @RequestLine(value = "GET /item?id={id}")
                String getItemCoupon(@Param("id") String id);
            }
        
            public static void main(String... args) {
                Fq target = Feign.builder()
                        .logger(new Slf4jLogger())
                        .logLevel(Logger.Level.FULL)
                        .target(Fq.class, "http://localhost");
                String itemCoupon = target.getItemCoupon(null);
        //结果 GET http://localhost/item HTTP/1.1
               
        ```

    * 如果是`+`或`/` 这种特殊字符

        ```java
            interface Fq {
                @RequestLine(value = "GET /item?id={id}")
                String getItemCoupon(@Param("id") String id);
            }
        
            public static void main(String... args) {
                Fq target = Feign.builder()
                        .logger(new Slf4jLogger())
                        .logLevel(Logger.Level.FULL)
                        .target(Fq.class, "http://localhost");
                String itemCoupon = target.getItemCoupon("a+b/c");
        //结果 GET http://localhost/item?id=a%2Bb%2Fc HTTP/1.1
        ```

        > 由上可以看出，出现在参数中的模板表达式，如果是空字符串则参数名会被保留下来，如果是null则移除参数名，如果有特殊字符，则进行转码。



* 如果是`@QueryMap`拼接查询参数

    ```java
     interface Fq {
            @RequestLine(value = "GET /item")
            String getItemCoupon(@QueryMap Map map);
        }
    
        public static void main(String... args) {
            Fq target = Feign.builder()
                    .logger(new Slf4jLogger())
                    .logLevel(Logger.Level.FULL)
                    .target(Fq.class, "http://localhost");
            Map<String, Object> map = new HashMap<>();
            map.put("a","1+1/2");
            map.put("b","");
            map.put("c",null);
    
            String itemCoupon = target.getItemCoupon(map);
            
    // 结果 GET http://localhost/item?a=1%2B1%2F2&b&c HTTP/1.1
            
    ```

    > 由上可以看出使用`RequestMap` 传递参数时，会对参数进行编码，且如果参数的值为null或空字符串，则保留参数的键

查看[Advanced Usage](https://github.com/OpenFeign/feign#advanced-usage) 有更多的示例。



> 如何处理斜杠 `/`
>
> 默认情况下`@RequestLine`不会 encode 斜杠，要改变这种默认行为，可以将`@ReqeustLine`的属性`decodeSlash`设置为`false`，参考这个[issus](https://github.com/OpenFeign/feign/issues/350),即默认情况下:
>
> ```java
> @RequestMapping("/foo/{id}")
> String getFooById(String id) {
> }
> myFeignClient.getFooById("/1/2/3")
> //真正请求地址将是 ‘/foo/1/2/3’，不会对/转码
>     
> ```



> 如何处理加号`+`
>
> 根据url规范，加号是允许出现在path和查询参数中的，但是现实中处理起来可能不一样，有一些老系统，会把`+`当成空格，但是现代的新系统在查询参数中不会将`+`当成空格，而是显示成`%2B`
>
> 如果你希望使用`+`作为空格，你可以直接只用` `字符来当成空格，或者编码成 `@2B`,
>
> 例如
>
> ```java
> @RequestMapping("/foo/{id}")
> String getFooById(String id) {
> }
> myFeignClient.getFooById("/1+1/2")
> //真正请求地址将是 ‘/foo/1+1/2’，不会对+转码
> ```
>
> #### 上面这说的+和/ 在路径中不会转码，但在参数中会被转码





#### Expander接口

`@Param`注解有一个可选的属性 `expander`允许控制该参数的解析，`expander`属性必须指向一个`Expander`接口的实现类，接口如下:

```java
public interface Expander {
    String expand(Object value);
}
```

##### 示例

```java
 interface Fq {
        @RequestLine(value = "GET /item?date={date}")
        String getItemCoupon(@Param(value = "date", expander = DateExpander.class) Date date);
    }

    //自定义格式化时间的expander
     public static class DateExpander implements Param.Expander {
        @Override
        public String expand(Object value) {
            if (!(value instanceof Date)) {
                throw new RuntimeException("不支持的类型");
            }
            Date date = (Date) value;
            return new SimpleDateFormat("yyyy-MM-dd").format(date);
        }
    }

    public static void main(String... args) {
        Fq target = Feign.builder()
                .logger(new Slf4jLogger())
                .logLevel(Logger.Level.FULL)
                .target(Fq.class, "http://localhost");

        String itemCoupon = target.getItemCoupon(new Date());
```

> 框架在调用此接口前判断参数是否为null，如果为null则按照上面规范忽略此参数的键值对，不再调用此接口，所以此接口的参数会被保证非null。
>
> 此接口的返回值可以是空字符串，也可以是null。返回值，按照上面[Undefined vs Empty Values](#Undefined vs Empty Values)规范处理。
>
> 返回值中的特殊字符会被pct-encoded处理，查看[Custom @Param Expansion](https://github.com/OpenFeign/feign#custom-param-expansion)有更多示例。



### 请求头扩展

和[请求参数扩展](#请求参数扩展)类似，但进行了如下修改：

* 默认不执行pct-encode ，（而请求参数扩展是默认encode的）。
* 未解析的表达式将被忽略，如果解析结果是空字符串，则这条请求头会被移除 （而请求参数空字符串会添加没有值的参数而不是忽略整个参数）

> 示例如下
>
> ```java
> public interface ContentService {
>   @RequestLine("GET /api/documents/{contentType}")
>   @Headers("Accept: {contentType}")
>   String getDocumentByType(@Param("contentType") String type);
> }
> ```



### 请求体扩展

`@Body`和[请求参数扩展](#请求参数扩展)类似，但进行了如下修改：

* 值不会进行encoder
* 不能解析的表达式将被忽略，（把模板当成字符串）。
* `Content-Type`头是必须要有的



***

## 定制化

Feign允许定制，它内置了一些可以实现的接口，通过`Feign.builder()`来实现定制，如定制自定义decoder：

```java
interface Bank {
  @RequestLine("POST /account/{id}")
  Account getAccountInfo(@Param("id") String id);
}

public class BankService {
  public static void main(String[] args) {
    Bank bank = Feign.builder().decoder(
        new AccountDecoder())
        .target(Bank.class, "https://api.examplebank.com");
  }
}
```



## 多种接口

这个例子中，target传递的不是（class，String） 而是一个target接口的实现类，上面传递的参数（class,String）会被构造为一个HardCodedTarget<T> (T class,String url) ，而你可以自己实现这一步。

```java
public class CloudService {
  public static void main(String[] args) {
    CloudDNS cloudDNS = Feign.builder()
      .target(new CloudIdentityTarget<CloudDNS>(user, apiKey));
  }
  
  class CloudIdentityTarget extends Target<CloudDNS> {
    /* implementation of a Target */
  }
}
```

> 使用自定义的target挺不错的，大家可以实现一个试试看。

***

## 整合示例

### Gson

[Gson](https://github.com/OpenFeign/feign/blob/master/gson)包含一个编码器一个解码器，和JSON API 一起工作

```java
public class Example {
  public static void main(String[] args) {
    GsonCodec codec = new GsonCodec();
    GitHub github = Feign.builder()
                         .encoder(new GsonEncoder())
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");
  }
}
```



### Jackson

[Jackson](https://github.com/OpenFeign/feign/blob/master/jackson) 使用Jackson来处理JSON也很好

```java
public class Example {
  public static void main(String[] args) {
      GitHub github = Feign.builder()
                     .encoder(new JacksonEncoder())
                     .decoder(new JacksonDecoder())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```



### OkHttp

底层使用[OkHttpClient](https://github.com/OpenFeign/feign/blob/master/okhttp)来处理发送请求，因为默认是使用java.net.URL connection来发送请求的，性能比较低

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .client(new OkHttpClient())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```



### Ribbon

[RibbonClient](https://github.com/OpenFeign/feign/blob/master/ribbon) 提供负载均衡，需要负载均衡的可以使用这个

```java
public class Example {
  public static void main(String[] args) {
    MyService api = Feign.builder()
          .client(RibbonClient.create())
          .target(MyService.class, "https://myAppProd");
  }
}
```



### Java 11 Http2

使用java11提供的Http2,高版本jdk使用这个是很好的

```java
GitHub github = Feign.builder()
                     .client(new Http2Client())
                     .target(GitHub.class, "https://api.github.com");
```



### Hystrix

[HystrixFeign](https://github.com/OpenFeign/feign/blob/master/hystrix)使用这个让Feign使用[Hystrix](https://github.com/Netflix/Hystrix)，带有熔断器功能

```java
public class Example {
  public static void main(String[] args) {
    MyService api = HystrixFeign.builder().target(MyService.class, "https://myAppProd");
  }
}
```



### SLF4J

[SLF4JModule](https://github.com/OpenFeign/feign/blob/master/slf4j) 这个组件允许让Feign底层使用SLF4j来记录日志

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .logger(new Slf4jLogger())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```



### Decoders

配置一个解码器来处理相应`Response`，默认的支持返回值为`String`,`byte[]`,`void`,可以看一下源码

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```



如果想让结果进入解码器前进行预处理，可以使用`mapAndDecode`方法，如下，传递解码器时传递一个lambda来预处理响应。

```java
public class Example {
  public static void main(String[] args) {
    JsonpApi jsonpApi = Feign.builder()
                         .mapAndDecode((response, type) -> jsopUnwrap(response, type), new GsonDecoder())
                         .target(JsonpApi.class, "https://some-jsonp-api.com");
  }
}
```



### Encoders

发送一个带请求体的post请求最简单的方法就是定义一个post请求，参数是一个不带任何注解的String或者byte[],然后不要忘了添加`Content-Type`请求头。

```java
interface LoginClient {
  @RequestLine("POST /")
  @Headers("Content-Type: application/json")
  void login(String content);
}

public class Example {
  public static void main(String[] args) {
    client.login("{\"user_name\": \"denominator\", \"password\": \"secret\"}");
  }
}
```



上面这么做很麻烦，你不得不手工处理json字符串，更简单的方法是定义一个`encoder`，如下编码器会将参数`Credentials`处理成json字符串。

```java
static class Credentials {
  final String user_name;
  final String password;

  Credentials(String user_name, String password) {
    this.user_name = user_name;
    this.password = password;
  }
}

interface LoginClient {
  @RequestLine("POST /")
  void login(Credentials creds);
}

public class Example {
  public static void main(String[] args) {
    LoginClient client = Feign.builder()
                              .encoder(new GsonEncoder())
                              .target(LoginClient.class, "https://foo.com");
    
    client.login(new Credentials("denominator", "secret"));
  }
}
```



### @Body templates

`@body`注解可以定义一个模板，从`@Param`标记的参数中提取数据，不要忘记添加`Content-Type`头

```java
interface LoginClient {

  @RequestLine("POST /")
  @Headers("Content-Type: application/xml")
  @Body("<login \"user_name\"=\"{user_name}\" \"password\"=\"{password}\"/>")
  void xml(@Param("user_name") String user, @Param("password") String password);

  @RequestLine("POST /")
  @Headers("Content-Type: application/json")
  // json 左右两边的括号必须转义的
  @Body("%7B\"user_name\": \"{user_name}\", \"password\": \"{password}\"%7D")
  void json(@Param("user_name") String user, @Param("password") String password);
}

public class Example {
  public static void main(String[] args) {
    client.xml("denominator", "secret"); // <login "user_name"="denominator" "password"="secret"/>
    client.json("denominator", "secret"); // {"user_name": "denominator", "password": "secret"}
  }
}
```



### @Headers

#### 设置请求头的api

`@Headers`注解可以设置在类上，以表示对所有接口生效，设置在方法上表示对该方法生效。

如果类中的所有方法均要添加这个请求头，就会很有用。

```java
@Headers("Accept: application/json")
interface BaseApi<V> {
  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String key, V value);
}
```



并且`@Header`注解是支持动态创建请求头的，从参数中提出数据：

```java
public interface Api {
   @RequestLine("POST /")
   @Headers("X-Ping: {token}")
   void post(@Param("token") String token);
}
```



如果请求头的键值都是动态的，而且还不知道具体有多少个，可以使用`@HeaderMap`注解处理

```java
public interface Api {
   @RequestLine("POST /")
   void post(@HeaderMap Map<String, Object> headerMap);
}
```



#### 为每个target设置请求头

有时候，如果同一个接口针对不同host（注:host是可以通过注入一个java.net.URI在运行中改变的）使用不同的请求头，或者请求头是根据具体请求变化的，则可以使用`RequestInterceptor` 或 `Target`来实现。



使用`RequestInterceptor`的例子在`RequestInterceptor`章节再讲。

下面是使用`Target`的解决灵活的请求头。

```java
 static class DynamicAuthTokenTarget<T> implements Target<T> {
    public DynamicAuthTokenTarget(Class<T> clazz,
                                  UrlAndTokenProvider provider,
                                  ThreadLocal<String> requestIdProvider);
    
    @Override
    public Request apply(RequestTemplate input) {
      TokenIdAndPublicURL urlAndToken = provider.get();
      if (input.url().indexOf("http") != 0) {
        input.insert(0, urlAndToken.publicURL);
      }
      input.header("X-Auth-Token", urlAndToken.tokenId);
      input.header("X-Request-ID", requestIdProvider.get());

      return input.request();
    }
  }
  
  public class Example {
    public static void main(String[] args) {
      Bank bank = Feign.builder()
              .target(new DynamicAuthTokenTarget(Bank.class, provider, requestIdProvider));
    }
  }
```



## 高级用法

### 基础Api

在很多情况下，服务端的接口遵循一致的约定，这时可以通过继承接口来处理。

示例

```java
interface BaseAPI {
  @RequestLine("GET /health")
  String health();

  @RequestLine("GET /all")
  List<Entity> all();
}
```

你可以使用继承，来获取父类中的方法

```java
interface CustomAPI extends BaseAPI {
  @RequestLine("GET /custom")
  String custom();
}
```



有些情况下，返回的Response表现形式也是一样的，如创建用户发送的json，获取用户得到的json，他们两个的形式都是一样的，可以定义公共泛型父类。

```java
@Headers("Accept: application/json")
interface BaseApi<V> {

  @RequestLine("GET /api/{key}")
  V get(@Param("key") String key);

  //获取时，将结果返序列化成V类型  
  @RequestLine("GET /api")
  List<V> list();

  //发送时，将V类型序列化
  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String key, V value);
}

interface FooApi extends BaseApi<Foo> { }

interface BarApi extends BaseApi<Bar> { }
```





### 日志



你可以设置一个`Logger` 和 `Logger.Level`来选择用什么处理日志以及日志级别

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .logger(new Logger.JavaLogger("GitHub.Logger").appendToFile("logs/http.log"))
                     .logLevel(Logger.Level.FULL)
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

> 注意：使用JavaLogger()时要避免使用无参的构造函数`JavaLogger()`来创建它，他有问题被弃用了，以后可能会删除。



### Request Interceptors

当你需要改变所有的请求，无论这个请求的地址是什么时，可以`Request Interceptors`。比如你想添加`X-Forwarded-For`请求头。



```java
static class ForwardedForInterceptor implements RequestInterceptor {
  @Override public void apply(RequestTemplate template) {
    template.header("X-Forwarded-For", "origin.host.com");
  }
}

public class Example {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new ForwardedForInterceptor())
                 .target(Bank.class, "https://api.examplebank.com");
  }
}
```



拦截器另一个常用的地方就是身份认证，如需要`BasicAuthRequestInterceptor`

```java
public class Example {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new BasicAuthRequestInterceptor(username, password))
                 .target(Bank.class, "https://api.examplebank.com");
  }
}
```



> 拦截器在Target前面执行，在Target的apply前一行代码。

### @Param扩展

上面讲到过`@Param`注解的expander 接口，下面这个例子将如果对参数进行修改，他将date格式化成毫秒值

```java
public interface Api {
  	@RequestLine("GET /?since={date}") 
    Result list(@Param(value = "date", expander = DateToMillis.class) Date date);
}
```

> 注意一点，如果有两个`@Param`标记的参数的名字是一样的，则后面的会覆盖前面的

### @QueryMap 动态查询参数

一个普通的map，添加上`@QueryMap`注解，它里面的值会被拿出来作为查询参数处理

```java
public interface Api {
  @RequestLine("GET /find")
  V find(@QueryMap Map<String, Object> queryMap);
}
```



不是map是pojo也是可以的，默认通过反射获取pojo内的字段拼接成查询参数。

也可以定义一个`QueryMapEncoder`来处理如何从pojo中拿值。

```java
public interface Api {
  @RequestLine("GET /find")
  V find(@QueryMap CustomPojo customPojo);
}
```

配置一个`queryMapEncoder`的例子。

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .queryMapEncoder(new MyCustomQueryMapEncoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

默认通过反射获取值，如果希望使用`java bean`规范通过 `get set`来获取值，可以配置一个`BeanQueryMapEncoder`

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .queryMapEncoder(new BeanQueryMapEncoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

> 注意：`@QueryMap`注解的解析在`@Requestline`之后，所以`@Requestline`上面定义的参数会被后来的`@QueryMap`定义的参数覆盖，且如果`@QueryMap`内定义一个空值参数会把`@Requestline`中定义的删掉。

### Error Handling

配置一个error handDecoder，所有不再2xx范围内的响应，都会被此handler的decode方法处理。

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .errorDecoder(new MyErrorDecoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

如果你想进行重试，可以抛出一个`RetryableException`，这样框架接收到了此异常调用注册的`Retryer`来处理重试。



### Retry 重试

默认情况下会重试所有的`IoException`，和ErrorHandling里面抛出的`RetryableException`,通过在builder是设置一个Retryer来定制这种行为

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .retryer(new MyRetryer())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```



`Retryer`通过方法`continueOrPropagate(RetryableException e)`来决定是否重试，无返回值。

如果允许重试，则直接返回。

如果不允许重试，请将参数中的RetryableException重新抛出。



框架为每个`Client`执行器clone一个`Retryer`,所以你可以在retryer上维护状态，而不用担心冲突。

>  注:为每个client创建Retryer的方式是调用Feign.build() 时传入的Retryer的clone()方法



如果不能重试成功，则抛出最后一次重试的`RetryException`,如果你想要导致异常的真正Exception，请使用

`exceptionPropagationPolicy()`构建Feign



### Options配置

可以在build Feign时设置默认的Options。

如果想为每个请求设置独立Options则，接口的参数如果有类型是`feign.Request.Options`类型的，会作为配置传入，存在多个的情况下只取第一个。



### 静态和默认方法

java8+ 支持接口的静态方法和默认方法，这样就允许Feign客户端包含逻辑，

比如你想提供默认参数，或者两个接口聚合成一个一个接口返回等



```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("GET /users/{username}/repos?sort={sort}")
  List<Repo> repos(@Param("username") String owner, @Param("sort") String sort);
	//提供默认值
  default List<Repo> repos(String owner) {
    return repos(owner, "full_name");
  }

  /**
   * 从repos()中拿到list，再循环调用contributors拿到信息，最后返回
   */
  default List<Contributor> contributors(String user) {
    MergingContributorList contributors = new MergingContributorList();
    for(Repo repo : this.repos(owner)) {
      contributors.addAll(this.contributors(user, repo.getName()));
    }
    return contributors.mergeResult();
  }

  static GitHub connect() {
    return Feign.builder()
                .decoder(new GsonDecoder())
                .target(GitHub.class, "https://api.github.com");
  }
}
```





### 一些内置类

#### feign.Response

```java
public final class Response implements Closeable {
	//状态码
  private final int status;
    //状态码后面的文字，如200 OK 则此字段就是OK
  private final String reason;
    //响应头
  private final Map<String, Collection<String>> headers;
  	//响应体
  private final Body body;
    //该响应对应的请求
  private final Request request;
  .
  .
  .
```



#### feign.Response.Body

```java
public interface Body extends Closeable {
	//数据长度
    Integer length();
	//是否允许重复读取
    boolean isRepeatable();
	//返回流
    InputStream asInputStream() throws IOException;
	
    @Deprecated
    default Reader asReader() throws IOException {
      return asReader(StandardCharsets.UTF_8);
    }
	//返回read流
    Reader asReader(Charset charset) throws IOException;
  }
```

