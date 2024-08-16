---
layout:     post
title:      "HttpClient Fluent API使用方法"
subtitle:   ""
date:       2019-06-21
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - java
---


# HttpClient Fluent API使用方法

### 1.加入 maven

``` maven

 <dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.8</version>
</dependency>
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>fluent-hc</artifactId>
    <version>4.5.8</version>
</dependency>

```



### 2.基本用法

```java

		Request.Get("http://www.baidu.com")
				.execute()
				.handleResponse(new ResponseHandler<String>() {
					@Override
					public String handleResponse(HttpResponse response) throws ClientProtocolException, IOException {
						return EntityUtils.toString(response.getEntity(), "utf-8");
					}
				});

```

### 3.额外参数

```java
			Request.Get("http://www.baidu.com")
					.connectTimeout(3000)
					.addHeader("key","value")
					.userAgent("test")
					.execute();

```


### 4.post

```java
			/*添加一个post参数*/
			BasicNameValuePair pair = new BasicNameValuePair("name","hh");
			Request.Post("http://www.baidu.com")
					.connectTimeout(3000)
					.bodyForm(pair)
					.execute();
```


### 5.post json

```java
		Request.Post("http://www.baidu.com")
					.connectTimeout(3000)
					.bodyString(JSON.toJSONString(var), ContentType.APPLICATION_JSON)
					.execute();

```

### 
```java
			Request.Post("https://www.baidu.com")
					.useExpectContinue()
					.version(HttpVersion.HTTP_1_1)
					.bodyString("啦啦啦", ContentType.DEFAULT_TEXT)
					.execute();

```