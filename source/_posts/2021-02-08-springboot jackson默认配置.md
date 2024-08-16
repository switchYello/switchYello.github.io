---
layout:     post
title:      "springboot jackson默认配置"
subtitle:   ""
date:       2021-02-08
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- jackson
typora-root-url: ..
---

​	

# springboot jackson默认配置



​		springboot是约定大约配置，默认就会进行比较合理的配置，自动配置jackson也是如此。下面看一下springboot是如何配置jackson的反序列化特性的。



## 配置jackson的方法

​		想要自定义配置jackson，只需实现`Jackson2ObjectMapperBuilderCustomizer`接口，并注册到spring容器里即可。

平时我们主要关注下面两个配置，一个是序列化时不返回值为null的字段，二是反序列化时遇到不认识的key不报错而是忽略此key。

```java
@Configuration
public class JacksonConfig implements Jackson2ObjectMapperBuilderCustomizer {

    @Override
    public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
        jacksonObjectMapperBuilder.serializationInclusion(JsonInclude.Include.NON_NULL);
        jacksonObjectMapperBuilder.failOnUnknownProperties(false);
    }

}
```



## 自动配置的jackson配置

如果没提供自定义的配置，将只使用springboot的默认配置，在`JacksonAutoConfiguration`类中，存在如下代码

```java
		@Bean
		@Primary
		@ConditionalOnMissingBean(ObjectMapper.class)
		public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
			return builder.createXmlMapper(false).build();
		}
```

可见其实使用`Jackson2ObjectMapperBuilder`类来创建ObjectMapper，且关闭了xml功能。





`Jackson2ObjectMapperBuilder`的创建同样在此类中，因为我们没有传入任何自定义`Jackson2ObjectMapperBuilderCustomizer`，所以其使用默认的值。

```java
		@Bean
		@ConditionalOnMissingBean(Jackson2ObjectMapperBuilder.class)
		public Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder(
				List<Jackson2ObjectMapperBuilderCustomizer> customizers) {
			Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
			builder.applicationContext(this.applicationContext);
			customize(builder, customizers);
			return builder;
		}
```



查看`Jackson2ObjectMapperBuilder`类内部的build方法源码，其也是创建一个新的ObjectMapper并对其进行配置，重点就在下面源码的`configure(mapper)`方法里面。

```java
	public <T extends ObjectMapper> T build() {
		ObjectMapper mapper;
		if (this.createXmlMapper) {
			mapper = (this.defaultUseWrapper != null ?
					new XmlObjectMapperInitializer().create(this.defaultUseWrapper) :
					new XmlObjectMapperInitializer().create());
		}
		else {
			mapper = new ObjectMapper();
		}
		configure(mapper);
		return (T) mapper;
	}
```



​		进入configure方法里，其将此构建器的属性进行非空判断，如果不为空就设置到ObjectMapper里。因为我们没有提供任何自定义配置，所以这些值都是空的，其中此方法里有一行调用了`customizeDefaultFeatures(objectMapper)`方法，其会设置默认配置。

```java
	public void configure(ObjectMapper objectMapper) {
		Assert.notNull(objectMapper, "ObjectMapper must not be null");

		if (this.findModulesViaServiceLoader) {
			// Jackson 2.2+
			objectMapper.registerModules(ObjectMapper.findModules(this.moduleClassLoader));
		}
		else if (this.findWellKnownModules) {
			registerWellKnownModulesIfAvailable(objectMapper);
		}

		if (this.modules != null) {
			for (Module module : this.modules) {
				// Using Jackson 2.0+ registerModule method, not Jackson 2.2+ registerModules
				objectMapper.registerModule(module);
			}
		}
		if (this.moduleClasses != null) {
			for (Class<? extends Module> module : this.moduleClasses) {
				objectMapper.registerModule(BeanUtils.instantiate(module));
			}
		}

		if (this.dateFormat != null) {
			objectMapper.setDateFormat(this.dateFormat);
		}
		if (this.locale != null) {
			objectMapper.setLocale(this.locale);
		}
		if (this.timeZone != null) {
			objectMapper.setTimeZone(this.timeZone);
		}

		if (this.annotationIntrospector != null) {
			objectMapper.setAnnotationIntrospector(this.annotationIntrospector);
		}
		if (this.propertyNamingStrategy != null) {
			objectMapper.setPropertyNamingStrategy(this.propertyNamingStrategy);
		}
		if (this.defaultTyping != null) {
			objectMapper.setDefaultTyping(this.defaultTyping);
		}
		if (this.serializationInclusion != null) {
			objectMapper.setSerializationInclusion(this.serializationInclusion);
		}

		if (this.filters != null) {
			objectMapper.setFilterProvider(this.filters);
		}

		for (Class<?> target : this.mixIns.keySet()) {
			objectMapper.addMixIn(target, this.mixIns.get(target));
		}

		if (!this.serializers.isEmpty() || !this.deserializers.isEmpty()) {
			SimpleModule module = new SimpleModule();
			addSerializers(module);
			addDeserializers(module);
			objectMapper.registerModule(module);
		}

		customizeDefaultFeatures(objectMapper);
		for (Object feature : this.features.keySet()) {
			configureFeature(objectMapper, feature, this.features.get(feature));
		}

		if (this.handlerInstantiator != null) {
			objectMapper.setHandlerInstantiator(this.handlerInstantiator);
		}
		else if (this.applicationContext != null) {
			objectMapper.setHandlerInstantiator(
					new SpringHandlerInstantiator(this.applicationContext.getAutowireCapableBeanFactory()));
		}
	}
```





​		可以看到当features里面不包含`FAIL_ON_UNKNOWN_PROPERTIES`时，即我们没有主动将`FAIL_ON_UNKNOWN_PROPERTIES`设为true或者false，springboot会将其设置为false，这也正符合我们平时的使用习惯。

```java
	private void customizeDefaultFeatures(ObjectMapper objectMapper) {
		if (!this.features.containsKey(MapperFeature.DEFAULT_VIEW_INCLUSION)) {
			configureFeature(objectMapper, MapperFeature.DEFAULT_VIEW_INCLUSION, false);
		}
        //列表内找不到该属性，就将此属性设置为false
		if (!this.features.containsKey(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)) {
			configureFeature(objectMapper, DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
		}
	}
```





## 总结

​		上面可以看出springboot会为我们自动配置好比较合理的配置，即约定大于配置，我们没有`FAIL_ON_UNKNOWN_PROPERTIES`的配置时，他会自动将其设为false。

​		于是我们在下次自定义配置时，不配置此属性也是可以的，将上面的配置改为下面这样，其也是能自动设置`FAIL_ON_UNKNOWN_PROPERTIES=false`的。

```java
@Configuration
public class JacksonConfig implements Jackson2ObjectMapperBuilderCustomizer {

    @Override
    public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
        jacksonObjectMapperBuilder.serializationInclusion(JsonInclude.Include.NON_NULL);
    }
}
```

​		但是考虑到不是每个人都看过源码，还是推荐加上，这样其他人看到会更清晰一点。

