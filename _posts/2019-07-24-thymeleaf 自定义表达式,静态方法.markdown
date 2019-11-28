---
layout:     post
title:      "thymeleaf 自定义表达式,静态方法"
subtitle:   ""
date:       2019-07-24
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - java
---


## thymeleaf 自定义表达式,静态方法


### 1.模板内置了一些方法，比如格式化时间的
	
	${#dates.format(article.date,'yyyy-MM-dd HH:mm')}
	
但有时候不满足我们的需求，需要自定义方法。


### 2.需求，遍历list，拼接list中对象的某个字段，以逗号分隔
	如：
	List<People> list = new ArrayList<>();
	list.add(new People(1,"小明"));
	list.add(new People(2,"小李"));
	list.add(new People(3,"小红"));
	list.add(new People(4,"小张"));

	现在想在页面上得到 “小明，小李，小红，小张” 这样拼接的字符串
	在内置函数是无法做到这一点的

### 3.解决方法

#### 3.1 写静态方法

```java

/*遍历集合，找到name对应的字段，并用 separator 拼接成字符串*/
public class IString {

	public String joinObjects(List<Object> list, String name, String separator) {
		List<String> tempList = new ArrayList<>(list.size());
		for (Object o : list) {
			try {
				String property = BeanUtils.getSimpleProperty(o, name);
				if (StringUtils.isNotBlank(property)) {
					tempList.add(property);
				}
			} catch (IllegalAccessException | InvocationTargetException | NoSuchMethodException e) {
				e.printStackTrace();
			}
		}
		return StringUtils.join(tempList.iterator(), separator);
	}
}
```



#### 3.2 写一个表达式配置工厂，这个类对于多个表达式可以通用


```java

public class IExpressionFactory implements IExpressionObjectFactory {

	//缓存下
	private Map<String, Object> map;

	public IExpressionFactory(Map<String, Object> map) {
		this.map = new HashMap<>(map);
	}

	/*获取表达式名字set集合*/
	@Override
	public Set<String> getAllExpressionObjectNames() {
		return map.keySet();
	}
	/*根据表达式名字获取表达式对象*/
	@Override
	public Object buildObject(IExpressionContext context, String expressionObjectName) {
		return map.get(expressionObjectName);
	}
	//是否允许缓存
	@Override
	public boolean isCacheable(String expressionObjectName) {
		return true;
	}
}

```	



#### 3.3 将上面的表达式工厂配置到springboot中

```java

@Component
public class IExpressionFactoryDialect extends AbstractDialect implements IExpressionObjectDialect {

	//表达式配置工厂
	private final IExpressionObjectFactory iExpressionFactory;

	public IExpressionFactoryDialect() {
		super("hcyExpression");
		//map中可以多个
		Map<String, Object> map = new HashMap<>();

		//配置我们刚才写的工具类，key为页面上引用的名字
		map.put("iStrings", new IString());
		iExpressionFactory = new IExpressionFactory(map);
	}

	/*返回工厂*/
	@Override
	public IExpressionObjectFactory getExpressionObjectFactory() {
		return iExpressionFactory;
	}

}

```


#### 3.4 使用


	${#iStrings.joinObjects(articles,'title',',') }

	页面上采用 #iStrings的方式引用该对象，传递参数（list，name，分隔符） ，可以获取获取list中对象的title字段 拼接得到的字符串