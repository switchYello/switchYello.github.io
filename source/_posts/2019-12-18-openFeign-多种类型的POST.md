---

layout:     post
title:      "Feign多种类型的POST"
subtitle:   "Feign NB"
date:       2019-12-18
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- openfeign
- Http客户端
typora-root-url: ..
---

# Feign多种类型的POST

* TOC
{:toc}
## 三种携带请求体的方式

前文[Feign高级功能](/2019/12/17/openFeign-高级功能/)说了，Feign有三种方式实现请求体，分别是

- 使用`@Body`注解
- 使用一个不加注解的参数
- 使用若干个添加`@Param`但未在`@ReqeustLine、@Heads...`等地方使用的参数



## POST的三种常见ContentType

而http的post有三种类型，分别是：

- application/json
- x-www.form-urlencoded
- form-data

下面按照这三种`ContentType`分别讨论一下。



## 1.发送`application/json`格式的请求体

发送此类post请求，需要添加头部`@Heads("Content-Type:application/json")`

### 使用`@Body`发送

>  注意使用`@Body`的方式构建请求体，别忘了注解中的值左右两遍的大括号需要转义。

```java
   interface TestJson {

        @RequestLine(value = "POST /test")
        @Body("%7B\"user_name\":\"{userName}\"%7D")
        @Headers("Content-Type:application/json")
        Response getItemCoupon(@Param("userName") String userName);
    }
```

### 使用不加注解的参数

```java
    interface TestJson2 {
        @RequestLine(value = "POST /test")
        @Headers("Content-Type:application/json")
        Response getItemCoupon(TestParam body);
    }
```

```java
        TestJson2 target = Feign.builder()
                .encoder(new JacksonEncoder())
                .target(TestJson2.class, "http://localhost:8081");

        target.getItemCoupon(new TestParam("username", "password"));
```

> 接口中有一个没有注解的参数`TestParam`，build时需要配置`JacksonEncoder`来处理它，将`TestParam`序列化成JSON字符串作为请求体。



### 使用没有使用的`@Param` 参数

```java
    interface TestJson3 {
        @RequestLine(value = "POST /test")
        @Headers("Content-Type:application/json")
        Response getItemCoupon(@Param("userName") String userName, @Param("password") String passWord);
    }
```

```java
 TestJson2 target = Feign.builder()
                .encoder(new JacksonEncoder())
                .target(TestJson2.class, "http://localhost:8081");

        target.getItemCoupon("username","password");
```

> 请看接口中有两个参数，但没有被使用，那么这两个参数就会组成一个`Map`,然后被配置的`JacksonEncoder`转成json字符串作为请求体。

### 结果报文

![image-20191218180418698](/img/in/2019-12-18-openFeign-多种类型的POST/image-20191218180418698.png)





##  2.发送`x-www-form-urlencoded`格式的post请求

### 使用`@Body`发送

```java
    interface TestWwwFormUrlencoded {
        @RequestLine(value = "POST /test")
        @Headers("Content-Type:application/x-www-form-urlencoded")
        @Body("name={name}&age={age}")
        Response getItemCoupon(@Param("name") String name, @Param("age") Integer age);
    }
```

```java
        TestWwwFormUrlencoded target = Feign.builder()
                .target(TestWwwFormUrlencoded.class, "http://localhost:8081");
        Response xiao = target.getItemCoupon(UriUtils.encode("小明"), 18);
	
```

> 这样的用法要注意两点，1.需要自己手工处理url编码,2.需要自己拼接参数，且数量有限。
>
> 虽然能写任意字符串拼接到body中，但是很麻烦不推荐这种用法。



### 使用不加注解的参数

```java
    interface TestWwwFormUrlencoded2 {
        @RequestLine(value = "POST /test")
        @Headers("Content-Type:application/x-www-form-urlencoded")
        Response getItemCoupon(String param);
    }
```

> 如果这样的话，传参需要自己拼接参数，还要自己url转码，很麻烦不推荐。



使用map来接收参数，并配合自定义encoder处理

```java
    interface TestWwwFormUrlencoded3 {
        @RequestLine(value = "POST /test")
        @Headers("Content-Type:application/x-www-form-urlencoded")
        Response getItemCoupon(Map<String, Object> params);
    }
```

```java
    public static class WwwEncode implements Encoder {

        @Override
        public void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException {
            Map<String, Object> map = (Map<String, Object>) object;
            List<String> list = new ArrayList<>();
            map.forEach((k, v) -> {
                list.add(UriUtils.encode(k) + "=" + UriUtils.encode(String.valueOf(v)));
            });
            String body = String.join("&", list);
            template.body(Request.Body.encoded(body.getBytes(), StandardCharsets.UTF_8));
        }
    }
```

> 这样比第一种方法方便点，但掉用时传递的参数是map，在不确定参数数目是很好用，但不利于识别，也不推荐。



### 使用没有使用的`@Param` 参数

```java
    interface TestWwwFormUrlencoded4 {
        @RequestLine(value = "POST /test")
        @Headers("Content-Type:application/x-www-form-urlencoded")
        Response getItemCoupon(@Param("name") String userName, @Param("age") String passWord);
    }
```



```java
    public static class WwwEncode implements Encoder {
        @Override
        public void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException {
            Map<String, Object> map = (Map<String, Object>) object;
            List<String> list = new ArrayList<>();
            map.forEach((k, v) -> {
                list.add(UriUtils.encode(k) + "=" + UriUtils.encode(String.valueOf(v)));
            });
            String body = String.join("&", list);
            template.body(Request.Body.encoded(body.getBytes(), StandardCharsets.UTF_8));
        }
    }
```

> Feign会把参数放到一个map里，传入`encoder`中。
>
> 同样需要encoder，，这种方法在确定参数数量的情况下比较好用，并且有参数名的提示，比上面那种方法用着舒服点，但要求固定的参数数量，多数情况下都是这样的。
>
> 并且可以同时使用`@QueryMap`注解，将多余的注解拼在url上，虽然不太好，但也是可以的。



### 结果报文

![image-20191218180217041](/img/in/2019-12-18-openFeign-多种类型的POST/image-20191218180217041.png)



## 3. 发送`form-data`格式的请求

> 这种格式主要用于上传文件，但Feign上传文件还是挺麻烦的，这里不讲上传文件的方法。如果服务端必须要求是`form-data`格式的请求，Feign也能实现，下面讲讲feign使用`form-data`发送post请求。



### form-data介绍

> 因为上面讲到的`x-www-form-urlencoded`形式的请求需要把字符进行url编码，这样体积一下就大了很多，而且不能用法发文件，所以`form-data`格式就被发明出来了。它使用一个约定好的符号进行分隔参数。报文如下

```http
POST /test HTTP/1.1
Host: localhost:8081
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Host: localhost:8081
Content-Length: 266

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="name"

abc
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="age"

18
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

> 上面的报文可以看出请求体部分是由boundary开始，由参数值结束。
>
> ![image-20191218181740787](/img/in/2019-12-18-openFeign-多种类型的POST/image-20191218181740787.png)
>
> boundary在请求头中约定好，且用于参数分隔的boundary要比请求头中定义的多了两个`--`。
>
> 最后收尾的boundary的尾部再多了两个`–-`
>
> 所以真正的请求体格式是
>
> ```java
> 设 bound = 111
> 
> --111
> Content-Disposition: form-data; name="name"
> 
> abc
> --111
> Content-Disposition: form-data; name="age"
> 
> 18
> --111--		//boundary比head中定义的多两个`--`，收尾的又在尾部多两个`--`
> ```



### 使用`@Body`发送

- 处理不了，在Body注解中拼接请求体太麻烦了，所以不采用这种方式。



### 使用不加注解的参数

```java
    interface TestFormData {
        @RequestLine(value = "POST /test")
        Response getItemCoupon(Map<String, Object> map);
    }
```

定义自定义encoder

```java
    public static class FormDataencodedPost implements Encoder {
        @Override
        public void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException {
            String boundary = "--------------" + System.currentTimeMillis();
            Map<String, Object> map = (Map<String, Object>) object;
            StringBuilder sb = new StringBuilder();
            map.forEach((k, v) -> {
                sb.append("--").append(boundary).append("\r\n");
                sb.append("Content-Disposition: form-data; name=\"").append(k).append("\"").append("\r\n");
                sb.append("\r\n");
                sb.append(v).append("\r\n");
            });
            sb.append("--").append(boundary).append("--\r\n");
            template.body(Request.Body.encoded(sb.toString().getBytes(StandardCharsets.UTF_8), StandardCharsets.UTF_8));

            template.removeHeader("Content-Type");
            template.header("Content-Type", "multipart/form-data; boundary=" + boundary);
        }
    }
```



```java
        TestFormData target = Feign.builder()
                .encoder(new FormDataencodedPost())
                .target(TestFormData.class, "http://localhost:8081");

        HashMap<String, Object> params = new HashMap<>();
        params.put("name", "老王");
        params.put("age", 12);

        target.getItemCoupon(params);
```

>- 参数是map，所以此处`encoder`中拿到的参数就是map，然后遍历map，将参数和值拼接成上面讲的`form-data`所需的形式。
>- boundary使用的是`-----`+`时间戳`的形式，也可以用其他随机生成的方式，for循环里面拼接boundary时，在每个boundary前面添加两个`-`，然后注意`\r\n`别遗漏了。
>- 参数循环拼接完成后在最后面在添加一个收尾的boundary，此boundary的前后各添加两个`-`。
>- 还要注意的一点时，先清空`Content-Type`再添加`Content-Type`，讲boundary设进去。



### 使用没有使用的`@Param` 参数

​	使用这个其实也是讲参数封装成map进行`encoder`和上面是一样的这里就不写了。



### 结果报文

> 
>
> ![image-20191218183647159](/img/in/2019-12-18-openFeign-多种类型的POST/image-20191218183647159.png)



`form-data`的拼接方式比较复杂，如果发送完后台接收不到数据，请仔细核对报文格式是否正确。可以打开`WireShark`抓包，再用其他工具进行正常的post，仔细比对两者报文的差异。



## 总结

以上就是使用Feign发送三种类型的POST请求的方法，业务上需要的场景一般就都能满足了，案例中用到的`encoder`写的比较简单，真实使用时请注意参数校验，处理好异常情况，或写更通用的`encoder`。