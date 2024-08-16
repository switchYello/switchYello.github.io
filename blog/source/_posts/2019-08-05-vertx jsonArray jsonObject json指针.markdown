---
layout:     post
title:      "vertx jsonArray jsonObject json指针"
subtitle:   ""
date:       2019-08-05
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - vertx
---


# vertx jsonArray jsonObject json指针

## JSON
	jsonobject 相当于一个Map<String,Object> , jsonArray相当于一个 List<Object>


## JsonObject
	构造方法，可以通过jsonString,构造，或者map构造。
```java

		String jsonStrng = "{\"name\":\"小明\"}";
		JsonObject j1 = new JsonObject(jsonStrng);
		
		Map<String,Object> map = new HashMap<>();
		map.put("name","小明");
		map.put("age","12");
		JsonObject j2 = new JsonObject(map);

```


#### 获取值 / 添加值 /键set /转成map

```java
String jsonStrng = "{\"name\":\"小明\",\"class\":\"小学一年级\",\"school\":{\"name\":\"xx小学\",\"address\":\"北京\"}}";
		JsonObject j1 = new JsonObject(jsonStrng);

		//获取值
		String name = j1.getString("name");
		System.out.println("获取值："+name);

		//获取值，为空则返回默认值
		String age = j1.getString("age", "空");
		System.out.println("获取值，为空则默认值"+age);

		//如果不提供默认值则返回null
		String age1 = j1.getString("age");
		System.out.println("获取值，为空则返回null:"+age1);

		//获取键的set集合
		Set<String> strings = j1.fieldNames();
		System.out.println("获取键的集合"+strings);

		//转成map
		Map<String, Object> map = j1.getMap();

//put 可以连续的调用put方法

	j1.put("foo","bar").put("num",123);


```

#### 合并json

```java

public static void main(String[] args) {
		/*
			{
				"name": "小明",
				"class": "小学一年级",
				"school": {
					"name": "xx小学",
					"address": "北京"
				}
			}
		 */

		String jsonStrng = "{\"name\":\"小明\",\"class\":\"小学一年级\",\"school\":{\"name\":\"xx小学\",\"address\":\"北京\"}}";
		JsonObject j1 = new JsonObject(jsonStrng);

		//将此json内的值，替换成提供的jsonObject的值
		String jsonStrng2 = "{\"name\":\"小明\",\"class\":\"小学一年级\",\"school\":{\"address\":\"安徽\"}}";
		JsonObject j2 = new JsonObject(jsonStrng2);
		JsonObject j3 = j1.mergeIn(j2);
		System.out.println("合并json1："+j3);
	}

/*

mergeIn(JsonObject obj);
mergeIn(JsonObject obj,Boolean deep);
mergeIn(JsonObject obj,int deep);
第一个表示浅合并，只要键相同则直接替换，
第二个如果参数传true，则表示深度合并，即如果某个键对应的值也是json，则对值内的json也进行合并。
第三个可以指定合并的深度，深度内为深合并，深度外为浅合并。

上面的代码，浅合并结果为
{"name":"小明","class":"小学一年级","school":{"address":"安徽"}}
深合并结果为
{"name":"小明","class":"小学一年级","school":{"name":"xx小学","address":"安徽"}}

*/

```	
#### 打印输出

	 encode 方法  正常打印出json字符串（一行）
	 encodePrettily方法 格式化打印json（更容易观察）


## JsonArray

#### 构造方法
```java
//通过字符串构造
	String json = "[1, 2, 3, 4]";
	JsonArray array = new JsonArray(json);

//通过list构造
	List<String> list = Arrays.asList("a", "b", "c", "d");
	JsonArray array = new JsonArray(list);

```

#### 基本操作，请查看api

## JsonPointer json指针

	json指针，类似于xml的xpath表达式，通过表达式可以直接获取对应位置的值。
	基本用法如下

```java

	String json = "{\"name\":\"小明\",\"class\":\"小学一年级\",\"school\":{\"name\":\"xx小学\",\"address\":\"北京\"}}";
	JsonObject entries = new JsonObject(json);


	JsonPointer append = JsonPointer.from("/school/address"); //创建指针，指定路径
	Object o = append.queryJson(entries);  //用该指针，查询上面的json
	System.out.println(o); //获取查询到的结果 ‘北京’


```