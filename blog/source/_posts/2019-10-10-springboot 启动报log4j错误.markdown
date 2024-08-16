---
layout:     post
title:      "springboot 启动报log4j错误"
subtitle:   ""
date:       2019-10-10
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
    - springboot
---


## springboot 启动报 `ERROR StatusLogger Log4j2 could not find a logging implementation. Please add log4j-core to the classpath. Using SimpleLogger to log to the console`

报错信息看起来与log4j有关，项目没打算用log4j，也没有主动引入log4j的，困扰了好久，以下是解决思路

```
springboot版本2.1.8
```



***


1. 查看maven 发现log4j-api包被 springboot-starter下的springboot-starter-loggering引入的。

2. 用idea ctrl+shift+f搜索报错信息，找到了报错日志点为 `org.apache.logging.log4j.LogManager`的静态代码块中，由于if语句`if (ProviderUtil.hasProviders())`判断为false，进入else语句打印出此log，猜测某个地方反射加载此类

3. 查看堆栈信息，找到`org.jboss.logging.LoggerProviders#findProvider`发现代码如下

```
        try {
            return tryJBossLogManager(cl, null);
        } catch (Throwable t) {

        }
        try {
            return tryLog4j2(cl, null);
        } catch (Throwable t) {

        }
        try {
            return tryLog4j(cl, null);
        } catch (Throwable t) {

        }
        try {
            Class.forName("ch.qos.logback.classic.Logger", false, cl);
            return trySlf4j(null);
        } catch (Throwable t) {

        }
        return tryJDK(null);
```

####代码去依次尝试获取jboss的log， log4j2， log4j， slf4j， jdklog来处理log.
#### 通过反射来加载`org.apache.logging.log4j.LogManager`时执行的上面静态代码块。

```
    private static LoggerProvider tryLog4j2(final ClassLoader cl, final String via) throws ClassNotFoundException {
        Class.forName("org.apache.logging.log4j.Logger", true, cl);
        Class.forName("org.apache.logging.log4j.LogManager", true, cl);
        Class.forName("org.apache.logging.log4j.spi.AbstractLogger", true, cl);
        LoggerProvider provider = new Log4j2LoggerProvider();
        logProvider(provider, via);
        return provider;
    }

```

### 查看报错点的if判断`if (ProviderUtil.hasProviders())`里面的代码如下

```

	protected static final String PROVIDER_RESOURCE = "META-INF/log4j-provider.properties";

    public static boolean hasProviders() {
        lazyInit();
        return !PROVIDERS.isEmpty();
    }

    protected static void lazyInit() {
    // noinspection DoubleCheckedLocking
    if (instance == null) {
        try {
            STARTUP_LOCK.lockInterruptibly();
            try {
                if (instance == null) {
                    instance = new ProviderUtil();
                }
            } finally {
                STARTUP_LOCK.unlock();
            }
        } catch (final InterruptedException e) {
            LOGGER.fatal("Interrupted before Log4j Providers could be loaded.", e);
            Thread.currentThread().interrupt();
        }
    }
}

    private ProviderUtil() {
        for (final LoaderUtil.UrlResource resource : LoaderUtil.findUrlResources(PROVIDER_RESOURCE)) {
            loadProvider(resource.getUrl(), resource.getClassLoader());
        }
    }

```
调用了ProviderUtil的构造方法，其中尝试获取配置文件`META-INF/log4j-provider.properties`但是获取为空
这样`PROVIDERS`这个list的值就为空，从而`hasProviders`方法返回false。

### 但是我的另一个项目启动时却不会打印出这样的错误log信息，按照上面的思路观察源码，发现另一个项目引入的是`log4j-api 2.11.1` 而报错的项目引入的是`log4j-api 2.8.1`，其中
### 2.11.1的ProviderUtil构造方法变成了这样

```
    private ProviderUtil() {
        for (final ClassLoader classLoader : LoaderUtil.getClassLoaders()) {
            try {
                loadProviders(classLoader);
            } catch (final Throwable ex) {
                LOGGER.debug("Unable to retrieve provider from ClassLoader {}", classLoader, ex);
            }
        }
        for (final LoaderUtil.UrlResource resource : LoaderUtil.findUrlResources(PROVIDER_RESOURCE)) {
            loadProvider(resource.getUrl(), resource.getClassLoader());
        }
    }

```

### 更高版本的log4j-api中适应多个classLoad尝试加载providers，在这里成功的找到了一个，随意if没有报错

## 观察maven的配置中并没有引入log4j的包，然后查看父项目的maven配置

```
<dependencyManagement>
    <dependencies>
   <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.8.1</version>
        </dependency>
</dependencies>
</dependencyManagement>

```

#根据maven的依赖加载原理，依赖关系近的会覆盖远的，所以父项目的依赖覆盖了springboot-start中的依赖，导致版本不一致。
#删除父项目中引入的依赖即可，问题原因找到