---
title: zuul实践之websocket支持
categories: 技术
tags: [java,springcloud,websocket]
---

## start

&emsp;&emsp;SpringCloud的zuul组件是解决java网关模块的大杀器,但是它依然是io阻塞模型,zuul2.0正在开源中,其io非阻塞特性以及对http2.0的支持和websocket的支持都是令人眼前一亮的闪光点.但是我们项目中现在依然使用zuul,目前该组件并未对websocket支持,目前我们项目中就需要网关加入websocket支持.

&emsp;&emsp;以前写过websocket的转发,当客户端与转发服务端建立连接时,同时让转发服务端维护一个client实例与目的服务端建立连接,通过这个client实例建立客户端与目的服务端消息转发的桥梁.编码难度其实不大,但是没必要重复造轮子,github上已经有大神提供了[zuul-websocket](https://github.com/mthizo247/spring-cloud-netflix-zuul-websocket)支持的框架,我们可以直接引入,通过配置修改达成我们的目的.


- - -

<!--more-->

- - -

## content

### 何为websocket

&emsp;&emsp;websocket是HTML5下一种新的协议.它实现了浏览器与服务器全双工通信,能更好的节省服务器资源和带宽并达到实时通讯的目的.它与HTTP一样通过已建立的TCP连接来传输数据,但是它和HTTP最大不同是：
- websocket是一种双向通信协议.在建立连接后,websocket服务器端和客户端都能主动向对方发送或接收数据,就像socket一样.
- websocket需要像TCP一样,先建立连接,连接成功后才能相互通信.

&emsp;&emsp;相对于传统HTTP每次请求-应答都需要客户端与服务端建立连接的模式,websocket是类似socket的TCP长连接通讯模式.一旦websocket连接建立后,后续数据都以帧序列的形式传输.在客户端断开websocket连接或server端中断连接前,不需要客户端和服务端重新发起连接请求.在海量并发及客户端与服务器交互负载流量大的情况下,极大的节省了网络带宽资源的消耗,有明显的性能优势,且客户端发送和接受消息是在同一个持久连接上发起,实时性优势明显.相比HTTP长连接,websocket有以下特点：
- 是真正的全双工方式,建立连接后客户端与服务器端是完全平等的,可以互相主动请求,而HTTP长连接基于HTTP,是传统的客户端对服务器发起请求的模式.
- HTTP长连接中,每次数据交换除了真正的数据部分外,服务器和客户端还要大量交换HTTP header,效率很低.websocket协议通过第一个request建立了TCP连接之后,之后交换的数据都不需要发送HTTP header就能交换数据,这显然和原有的HTTP协议有区别所以它需要对服务器和客户端都进行升级才能实现(主流浏览器都已支持HTML5).此外还有multiplexing、不同的URL可以复用同一个WebSocket连接等功能.这些都是HTTP长连接不能做到的.

### Srping websocket

&emsp;&emsp;Spring框架对websocket提供了基于`STOMP`协议(可用于广播)和非`STOMP`协议(一般用于点对点通信)的实现.非`STOMP`协议的websocket较为简单,这里不做过多介绍,网关使用基于`STOMP`协议的websocket转发.

&emsp;&emsp;基于`STOMP`协议的websocket属于发布-订阅模型.当客户端与服务端建立连接后,客户端向指定路由发送消息,服务端接收消息进行处理后,会把响应消息转发给另一路由,所有与服务端建立连接且订阅该路由的客户端都会接收到该消息.spring websocket配置如下:
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketStompConfig extends AbstractWebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");  //订阅路由前缀
        config.setApplicationDestinationPrefixes("/send");  //发送路由前缀
//        config.setUserDestinationPrefix("/user");  //与@SendToUser配合使用,点对点通信订阅路由前缀
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/websocket").setAllowedOrigins("*").withSockJS();  //连接路由,开启SockJS支持
    }
}
```
```java
@Controller
public class WebsocketMessageController {

    @MessageMapping("/message")
//    @SendTo(value = "/topic/message") 若不配置改标签订阅路径为:订阅路由前缀(/topic)+"/message"
//    @SendToUser(value = "/message", broadcast = false)
    public String sendMessage(String message) throws Exception {
        System.out.println("receive message:" + message);
        return message;
    }
}
```
```java
public class WebsocketStompClient {

    static public class MyStompSessionHandler
        extends StompSessionHandlerAdapter {

        private String userId;

        private String wsSendUri;

        private String wsSubscribeUri;

        public MyStompSessionHandler(String userId, String wsSendUri, String wsSubscribeUri) {
            this.userId = userId;
            this.wsSendUri = wsSendUri;
            this.wsSubscribeUri = wsSubscribeUri;
        }

        private void showHeaders(StompHeaders headers) {
            for (Map.Entry<String, List<String>> e : headers.entrySet()) {
                System.err.print("  " + e.getKey() + ": ");
                boolean first = true;
                for (String v : e.getValue()) {
                    if (!first) {
                        System.err.print(", ");
                    }
                    System.err.print(v);
                    first = false;
                }
                System.err.println();
            }
        }

        private void sendJsonMessage(StompSession session) {
            String msg = String.format("hello from %s", userId);
            session.send(wsSendUri, msg);
        }

        private void subscribeTopic(String topic, StompSession session) {
            session.subscribe(topic, new StompFrameHandler() {

                @Override
                public Type getPayloadType(StompHeaders headers) {
                    return String.class;
                }

                @Override
                public void handleFrame(StompHeaders headers,
                    Object payload) {
                    System.err.println(
                        String.format("%s receive message: %s", userId, payload.toString()));
                }
            });
        }

        @Override
        public void afterConnected(StompSession session,
            StompHeaders connectedHeaders) {
            System.err.println("Connected! Headers:");
            showHeaders(connectedHeaders);

            subscribeTopic(wsSubscribeUri, session);
            sendJsonMessage(session);
        }
    }

    public static void startClient(String wsConnectUrl, String wsSendUri, String wsSubscribeUri,
        long sleepForSecond)
        throws Exception {
        WebSocketClient simpleWebSocketClient =
            new StandardWebSocketClient();
        List<Transport> transports = new ArrayList<>(1);
        transports.add(new WebSocketTransport(simpleWebSocketClient));

        SockJsClient sockJsClient = new SockJsClient(transports);
        WebSocketStompClient stompClient =
            new WebSocketStompClient(sockJsClient);
        stompClient.setMessageConverter(new StringMessageConverter());

        String userId = "Mr.J---" + new Random().nextInt(1000);
        StompSessionHandler sessionHandler = new MyStompSessionHandler(userId, wsSendUri,
            wsSubscribeUri);
        StompSession session = stompClient.connect(wsConnectUrl, sessionHandler)
            .get();
        Long i = 0L;
        while (true) {
            TimeUnit.SECONDS.sleep(sleepForSecond);
            String msg = String.format("%s send %s !", userId, i++);
            session.send(wsSendUri, msg);
        }
    }

    public static void startClient(String wsConnectUrl, String wsSendUri, String wsSubscribeUri)
        throws Exception {
        startClient(wsConnectUrl, wsSendUri, wsSubscribeUri, 1L);
    }
}
```

### Zuul websocket

&emsp;&emsp;zuul增加websocket支持,直接参照github上大神的框架[spring-cloud-netflix-zuul-websocket](https://github.com/mthizo247/spring-cloud-netflix-zuul-websocket)配置,这里不做过多介绍.该项目可通过maven直接引入依赖,经过测试,确实可以从eureka获取了服务地址列表建立连接,但是并不会采用轮询策略,每次建立websocket连接时都只会使用固定的服务地址.

&emsp;&emsp;该框架不支持非`STOMP`协议的websocket,也不支持`@SendToUser`配置,所以想要实现非广播模式,而是点对点的websocket通信只能自己造轮子,或者服务端发送路由中加上`UUID`,`@MessageMapping("/message/{uuid}")`,客户端建立连接时需要生成自己唯一的`UUID`,并向该路径发送和订阅消息以保证点对点通信.(当然这种需要客户端的支持的方案并不好,我们项目太紧,没有时间造轮子,所以我只好选用这种退而求其次的方案啦).

&emsp;&emsp;经测试,使用该框架时,在收到订阅路由的消息时,消息不能被正确的`MessageConverter`转换,会发生客户端无法收到订阅路由的消息的情况.(不知道是不是我代码的问题,但是断点调试时确实发现框架里有点问题).遇到这种情况,可以自己声明`MessageConverter`解决.
```java
    @Bean
    public WebSocketStompClient stompClient(ThreadPoolTaskScheduler taskScheduler) {
        int bufferSizeLimit = 8388608;
        StandardWebSocketClient webSocketClient = new StandardWebSocketClient();
        ArrayList transports = new ArrayList();
        transports.add(new WebSocketTransport(webSocketClient));
        SockJsClient sockJsClient = new SockJsClient(transports);
        WebSocketStompClient client = new WebSocketStompClient(sockJsClient);
        client.setInboundMessageSizeLimit(bufferSizeLimit);
        // 自定义MessageConverter
        client.setMessageConverter(messageConverter());
        client.setTaskScheduler(taskScheduler);
        client.setDefaultHeartbeat(new long[]{0L, 0L});
        return client;
    }

    /**
     * 自定义MessageConverter
     *
     * @return MessageConverter
     */
    private MessageConverter messageConverter() {
        return new MessageConverter() {

            private MessageConverter messageConverter = new StringMessageConverter();

            @Override
            public Object fromMessage(Message<?> message, Class<?> aClass) {
                if (aClass == Object.class) {
                    aClass = String.class;
                }
                return messageConverter.fromMessage(message, aClass);
            }

            @Override
            public Message<?> toMessage(Object o, MessageHeaders messageHeaders) {
                return messageConverter.toMessage(o, messageHeaders);
            }
        };
    }
```
&emsp;&emsp;一般网关都会做token的效验用来拦截请求,zuul的过滤器可以实现HTTP协议的请求拦截,但是websocket如何拦截这些非法连接请求呢?使用该框架确实没看到这方面的拦截支持,仔细看了下该框架的配置类,也确实不好直接扩展.只能万般无奈选用了下下策(用别人造好的轮子总是各种无奈啊),完整重写该配置类,禁用原来的配置类.这里只贴下websocket拦截相关的代码.
```java
 	@Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        for (Map.Entry<String, ZuulWebSocketProperties.WsBrokerage> entry : zuulWebSocketProperties
            .getBrokerages().entrySet()) {
            ZuulWebSocketProperties.WsBrokerage wsBrokerage = entry.getValue();
            if (wsBrokerage.isEnabled()) {
                registry.addEndpoint(wsBrokerage.getEndPoints())
                    // set HandShakeHandler
                    .setHandshakeHandler(new CustomHandshakeHandler())
                    .setAllowedOrigins("*").withSockJS();
            }
        }
    }
    
    /**
     * 自定义HandshakeHandler,用于拦截请求(这里不是我故意继承那么多看似无关的接口,是必须要继承哦!!!)
     */
    class CustomHandshakeHandler implements HandshakeHandler, ServletContextAware, Lifecycle {

        private DefaultHandshakeHandler defaultHandshakeHandler = new DefaultHandshakeHandler();

        @Override
        public boolean doHandshake(ServerHttpRequest request, ServerHttpResponse response,
            WebSocketHandler wsHandler, Map<String, Object> attributes)
            throws HandshakeFailureException {
            //自己写拦截逻辑,如是非法连接请求直接return false;
            return true;
        }

        @Override
        public void start() {
            defaultHandshakeHandler.start();
        }

        @Override
        public void stop() {
            defaultHandshakeHandler.stop();
        }

        @Override
        public boolean isRunning() {
            return defaultHandshakeHandler.isRunning();
        }

        @Override
        public void setServletContext(ServletContext servletContext) {
            defaultHandshakeHandler.setServletContext(servletContext);
        }
    }
```

### end

&emsp;&emsp;这次实践给我最大的感受就是造轮子什么的还是自己造的用起来才比较舒服,别人造的轮子总感觉有些方面不能满足自己的需求.这也是告诉大家,在造轮子的时候要多考虑框架的可扩展性哦!