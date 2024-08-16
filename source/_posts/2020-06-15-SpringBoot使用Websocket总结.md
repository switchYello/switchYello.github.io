---



layout:     post
title:      "SpringBoot使用Websocket总结"
subtitle:   ""
date:       2020-06-15
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- java
- spring
- websocket
- springboot
---



# SpringBoot使用Websocket总结



## 1.添加依赖

```xml
      <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
```



## 2.写处理器

写一个类实现`org.springframework.web.socket.WebSocketHandler`，

更推荐继承`org.springframework.web.socket.handler.AbstractWebSocketHandler`类。



```java
public class TestHandler extends AbstractWebSocketHandler {

    //处理textmessage
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        super.handleTextMessage(session, message);
    }

    //处理binarymessage
    @Override
    protected void handleBinaryMessage(WebSocketSession session, BinaryMessage message) throws Exception {
        super.handleBinaryMessage(session, message);
    }

    //处理pong
    @Override
    protected void handlePongMessage(WebSocketSession session, PongMessage message) throws Exception {
        super.handlePongMessage(session, message);
    }

    //报错时
    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        super.handleTransportError(session, exception);
    }

    //连接关闭时
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        super.afterConnectionClosed(session, status);
    }

    //连接成功时
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        super.afterConnectionEstablished(session);
    }

    //是否支持报文拆包
    @Override
    public boolean supportsPartialMessages() {
        return super.supportsPartialMessages();
    }
}
```





## 3.配置路径和拦截器

​		在一个`Configuration`类上，添加`EnableWebSocket`注解启动`Websocket`。

​		注入`WebSocketConfigurer`的`Bean`即可启动`websocket`服务端。

​		`addInterceptors`方法是添加拦截器，这个拦截器拦截在`http`协议转向`websocket`的那个请求上，如果底层使用的是`servlet`可以强转成`ServletServerHttpRequest`，在这里能拿到`websocket`连接时携带的`param`和`head`。

```java
@EnableWebSocket
@Configuration
public class WebsocketConfig {

    private static Logger log = LoggerFactory.getLogger(WebsocketConfig.class);

    @Bean
    public WebSocketConfigurer wsc() {
        return registry -> registry.addHandler(new TestHandler(), "/websocket/test")
                .addInterceptors(new HandshakeInterceptor() {
                    @Override
                    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
                        ServletServerHttpRequest req = (ServletServerHttpRequest) request;
                        return true;
                    }

                    @Override
                    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception exception) {

                    }
                }).setAllowedOrigins("*");
    }


}

```





## 4.技巧

 1. 拦截器返回`false`，则不会进行`websocket`协议的转换，如必须要登录才能连接的情况，可以在拦截器里拿到请求`session_id`来判断,也可以获取参数判断。

 2. 拦截器里的第三个参数，是一个`map`，放入此`map`的值可以从`WebSocketHandler`里面的`WebSocketSession`身上拿出来。

 3. `supportsPartialMessages`回调方法是决定是否接受半包，因为`websocket`协议比较底层，好像`Tcp`协议一样，如果发送大消息可能会拆成多个小报文。如果不希望处理不完整的报文，希望底层帮忙聚合成完整消息将此方法返回`false`,这样底层会等待完整报文到达聚合后才回调。

 4. `WebSocketSession`身上能设置最大报文大小，如果报文过大则会报错的,可以将限制调大。可以在`afterConnectionEstablished`方法里对`WebSocketSession`进行设置。`tomcat`的`session`默认大小就是     `8*1024`。

    ```java
     @Override
        public void afterConnectionEstablished(WebSocketSession session) throws Exception {
            session.setBinaryMessageSizeLimit(8 * 1024);
            session.setTextMessageSizeLimit(8 * 1024);
        }
    ```

	5. `WebSocketSession`是不支持并发发送消息的，要么自己做同步，要么包装`WebSocketSession`为`ConcurrentWebSocketSessionDecorator`，关于`ConcurrentWebSocketSessionDecorator`的使用可以参考[Spring并发发送Websocket消息](/2020/06/07/Spring并发发送Websocket消息/)。

	6. `setAllowedOrigins("*")`为控制跨域的，测试阶段可以设成`*`,不然不允许跨域链接`Websocket`。

	7. 百度搜`在线websocket`，有在线连接`Websocket`的网站，这样后端自己做测试比较方便。

    

## 5.原理

`EnableWebSocket`引入了`DelegatingWebSocketConfiguration`。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebSocketConfiguration.class)
public @interface EnableWebSocket {
}
```

​	

先看`DelegatingWebSocketConfiguration`这个类。首先类被构造后，会自动注入`WebSocketConfigurer`，这个就是我们配置的`Bean`。

```java
@Autowired(required = false)
	public void setConfigurers(List<WebSocketConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.configurers.addAll(configurers);
		}
	}
```



然后父类会注入一个`HandlerMapping`，会创建一个`ServletWebSocketHandlerRegistry`，调用所有的

`configurer.registerWebSocketHandlers(registry);`方法进行配置。

最后从`registry.getHandlerMapping()`获取`HandlerMapping`，这个`HandlerMapping`最终会注册到`Springmvc`里，接受到请求时会使用该`HandlerMapping`来处理请求。

```java
	@Bean
	public HandlerMapping webSocketHandlerMapping() {
		ServletWebSocketHandlerRegistry registry = initHandlerRegistry();
		if (registry.requiresTaskScheduler()) {
			TaskScheduler scheduler = defaultSockJsTaskScheduler();
			Assert.notNull(scheduler, "Expected default TaskScheduler bean");
			registry.setTaskScheduler(scheduler);
		}
        //使用我们的配置，构造出HandlerMapping
		return registry.getHandlerMapping();
	}
```



```java
	private ServletWebSocketHandlerRegistry initHandlerRegistry() {
		if (this.handlerRegistry == null) {
			this.handlerRegistry = new ServletWebSocketHandlerRegistry();
			registerWebSocketHandlers(this.handlerRegistry);
		}
		return this.handlerRegistry;
	}
```

```java
	@Override
	protected void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
		for (WebSocketConfigurer configurer : this.configurers) {
			configurer.registerWebSocketHandlers(registry);
		}
	}
```





下面是获取`HandlerMapping`的过程

​			这里会创建一个`WebSocketHandlerMapping`，并设置`urlMap`，`urlMap`里面存放的是url 和对应的`HttpRequestHandle`。

```java
	public AbstractHandlerMapping getHandlerMapping() {
		Map<String, Object> urlMap = new LinkedHashMap<>();
		for (ServletWebSocketHandlerRegistration registration : this.registrations) {
			MultiValueMap<HttpRequestHandler, String> mappings = registration.getMappings();
			mappings.forEach((httpHandler, patterns) -> {
				for (String pattern : patterns) {
					urlMap.put(pattern, httpHandler);
				}
			});
		}
        
		WebSocketHandlerMapping hm = new WebSocketHandlerMapping();
		hm.setUrlMap(urlMap);
		hm.setOrder(this.order);
		if (this.urlPathHelper != null) {
			hm.setUrlPathHelper(this.urlPathHelper);
		}
		return hm;
	}	
```



这里是将我们的配置构建成一个`WebSocketHttpRequestHandler`来处理请求，其中包含了`路径，拦截器，WebSocketHandler，HandshakeInterceptor，HandshakeHandler`。

```java
	@Override
	protected void addWebSocketHandlerMapping(MultiValueMap<HttpRequestHandler, String> mappings,
			WebSocketHandler webSocketHandler, HandshakeHandler handshakeHandler,
			HandshakeInterceptor[] interceptors, String path) {

		WebSocketHttpRequestHandler httpHandler =
				new WebSocketHttpRequestHandler(webSocketHandler, handshakeHandler);

		if (!ObjectUtils.isEmpty(interceptors)) {
			httpHandler.setHandshakeInterceptors(Arrays.asList(interceptors));
		}
		mappings.add(httpHandler, path);
	}
```



`HandshakeHandler`这个类也是可以自定义的，他与协议转换有关，spring会自动根据底层实现来决定这个是什么，一般不要自己写。



​	`HttpRequestHandle`处理过程如下，先调用拦截器的before，再掉用`HandshakeHandler`做协议转换，再调用拦截器的after。这里调用拦截器前创建的Map `attributes`，最终会设置到`WebsocketSession`上

```java
@Override
	public void handleRequest(HttpServletRequest servletRequest, HttpServletResponse servletResponse)
			throws ServletException, IOException {

		ServerHttpRequest request = new ServletServerHttpRequest(servletRequest);
		ServerHttpResponse response = new ServletServerHttpResponse(servletResponse);

		HandshakeInterceptorChain chain = new HandshakeInterceptorChain(this.interceptors, this.wsHandler);
		HandshakeFailureException failure = null;

		try {
			if (logger.isDebugEnabled()) {
				logger.debug(servletRequest.getMethod() + " " + servletRequest.getRequestURI());
			}
			Map<String, Object> attributes = new HashMap<>();
            //调用拦截器
			if (!chain.applyBeforeHandshake(request, response, attributes)) {
				return;
			}
            //处理握手
			this.handshakeHandler.doHandshake(request, response, this.wsHandler, attributes);
            //再调用拦截器
			chain.applyAfterHandshake(request, response, null);
		}
		catch (HandshakeFailureException ex) {
			failure = ex;
		}
		catch (Throwable ex) {
			failure = new HandshakeFailureException("Uncaught failure for request " + request.getURI(), ex);
		}
		finally {
			if (failure != null) {
				chain.applyAfterHandshake(request, response, failure);
				response.close();
				throw failure;
			}
			response.close();
		}
	}
```





下面是协议升级前做的操作，创建`StandardWebSocketSession`时将`attr`也就是那个拦截器里的`map`设置进去。

```java
@Override
	public void upgrade(ServerHttpRequest request, ServerHttpResponse response,
			@Nullable String selectedProtocol, List<WebSocketExtension> selectedExtensions,
			@Nullable Principal user, WebSocketHandler wsHandler, Map<String, Object> attrs)
			throws HandshakeFailureException {

		HttpHeaders headers = request.getHeaders();
		InetSocketAddress localAddr = null;
		try {
			localAddr = request.getLocalAddress();
		}
		catch (Exception ex) {
			// Ignore
		}
		InetSocketAddress remoteAddr = null;
		try {
			remoteAddr = request.getRemoteAddress();
		}
		catch (Exception ex) {
			// Ignore
		}
		//创建此类时将attrs作为参数传进去
		StandardWebSocketSession session = new StandardWebSocketSession(headers, attrs, localAddr, remoteAddr, user);
		StandardWebSocketHandlerAdapter endpoint = new StandardWebSocketHandlerAdapter(wsHandler, session);

		List<Extension> extensions = new ArrayList<>();
		for (WebSocketExtension extension : selectedExtensions) {
			extensions.add(new WebSocketToStandardExtensionAdapter(extension));
		}

		upgradeInternal(request, response, selectedProtocol, extensions, endpoint);
	}
```





## Springmvc处理过程

​		在`DispatcherServlet`里面，`websocket`升级的请求是由`HttpRequestHandlerAdapter`处理的，这个类是在

`org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport#httpRequestHandlerAdapter`方法里注册进来的。

处理过程也很简单,直接强转调用。

```java
	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		((HttpRequestHandler) handler).handleRequest(request, response);
		return null;
	}
```





## 完