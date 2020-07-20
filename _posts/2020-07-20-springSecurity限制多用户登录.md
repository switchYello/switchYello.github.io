---
layout:     post
title:      "springSecurity限制多用户登录"
subtitle:   ""
date:       2020-07-20
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- springboot
- springSecurity
- AuthenticationManager
typora-root-url: ..
---



# springSecurity限制多用户登录

​	

​		限制用户多次登录的原理就是，每次用户登录时，将用户的`sessionId`存储起来，下次在登录时检查已经存在的`sessionId`列表里此用户登陆了几次，就能据此做处理了。



​		最简单的就像这样设置，这样做以后后面登录的会将前面最近未使用的session挤掉。

```java
   http.sessionManagement().maximumSessions(1);
```

​	

​		如果想要禁止后面人的登录，而不是挤掉前面的人，可以这么做。可以拒绝后面登录的请求。这么做前面的人只是关闭浏览器而没有退出，也是处于在线状态的，后面人也是无法登录的。

```java
http.sessionManagement().maximumSessions(1).maxSessionsPreventsLogin(true);
```



​		但是还有一个问题，被挤掉的人的session信息仍然存储在`SessionRegistry`里，需要配置一个监听器，监听到session死亡时将其删除。这里只需要配置一个监听器即可。

​		这个监听器在监听到session死亡时，会发送事件通知。

```java
  @Bean
    public HttpSessionEventPublisher httpSessionEventPublisher(){
        return new HttpSessionEventPublisher();
    }

```



​		默认被挤掉的用户再次访问时，会通过write输出一段话，下面这样：

```java
@Override
public void onExpiredSessionDetected(SessionInformationExpiredEvent event)
    throws IOException {
    HttpServletResponse response = event.getResponse();
    response.getWriter().print(
        "This session has been expired (possibly due to multiple concurrent "
        + "logins being attempted as the same user).");
    response.flushBuffer();
}
```

​	

​		这显然不是我们想要的效果，我么想输出一段`json`或者重定向到首页这样的操作。那么可以这样配置都可以。下面两种配置方式中选择一个。

```java
    //自定义处理器   
	http.sessionManagement().maximumSessions(1).expiredSessionStrategy(new SessionInformationExpiredStrategy() {
            @Override
            public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
                HttpServletResponse response = event.getResponse();
                response.setContentType("application/json");
                response.getWriter().write("{\"status\":\"200\"}");
            }
        });

//或者配置一个链接， 将会重定向到此链接
        http.sessionManagement().maximumSessions(1).expiredUrl("/home");
        
```



​		

​		会话固定攻击导致的问题

​		为了防止会话固定攻击，每次登录完成后会改变`sessionId`。这将导致一个问题，如果在登录状态下，再次调用登录接口，`SessionRegistry`里会将新登录的`sessionId`记录下来，但不会删除旧登录的`sessionId`，因为旧的session只是改Id了，并没有销毁。

解决这个问题可以使用下面的办法。配置会话固定攻击的策略为`migrateSession`。这样会创建一个新的session，并将旧session里的内容复制进来。这样做会触发session销毁的监听器。

```java
        http.sessionManagement().maximumSessions(1)
                .and()
                .sessionFixation().migrateSession();
```



























































