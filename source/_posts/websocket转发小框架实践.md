---
title: websocket转发小框架实践
categories: 技术
tags: [java,springboot,springcloud,websocket]
---

## start

&emsp;&emsp;之前写过介绍网上大神写的基于stomp协议的zuul-websocket组件,通过该组件支持网关websocket代理转发.但是我们在项目中的使用场景往往是非stomp协议的websocket转发,所以那个组件可能就无法直接使用啦.由于项目开发任务很重,时间也很紧,一直犹豫要不要写一个类似的websocket转发的小框架.犹豫再三,最终花了几天时间写完啦.源码请查看[spring-websocket-forward](https://github.com/ji4597056/websocket-forward).

- - -

<!--more-->

- - -

## content

### 基本目标

- server端打算直接使用`spring-boot-starter-websocket`.(重复造轮子没有任何意义)
- 通过`application.yml`读取websocket配置,自动注册路由和对应的处理器
- client使用netty websocket client.
- 集成spring cloud注册中心(Eureka,Consul,Zookeeper),通过service id获取服务地址列表
- websocket转发使用地址轮询策略
- 需要灵活配置处理器,拦截器
- 需要高扩展性

### 具体实现

&emsp;&emsp;由于github可以看到项目源码,所以不打算把代码都贴一遍,大概说一下设计思路吧.

- **server端路由配置**

&emsp;&emsp;详见`WsForwardProperties`.
&emsp;&emsp;参考`zuul`的作法,配置时提供属性`serviceId`和`listOfServers`,`serviceId`主要用于从注册中心获取转发服务地址列表,当未设置该属性时,可以设置`listOfServers`直接获取具体转发服务地址列表.
&emsp;&emsp;由于`serverId`需要`spring cloud`相关注册中心组件支持,故引入`spring-cloud-commons`,其`DiscoveryClient`为注册中心获取服务地址的核心接口.
&emsp;&emsp;为了配置路由更加自由,并且不希望该组件仅仅为了弥补`zuul`对于websocket转发支持,所以提供了`prefix`作为服务端路由前缀以及`forwardPrefix`作为转发路由前缀.
&emsp;&emsp;对于转发地址列表的选取策略,就简单使用轮询策略,如果需要多种策略支持,可能需要`ribbon`组件的支持,暂时不打算依赖太多其他类库.

- **server和client的session交互**

&emsp;&emsp;详见`AbstractWsServerHandler`.
&emsp;&emsp;server和client的session交互使用简单的循环引用,server和client分别保存对方的session,由于每次消息交互都是不同线程,所以server端不能使用`ThreadLocal`保存session,这里使用`Map`保存session.(key:sever session id, value:client channel).
&emsp;&emsp;当server端建立连接成功后,随即实例化client选取地址进行连接,并把server的session传递给client,这里可能有性能问题,需要进行优化.
&emsp;&emsp;当出现异常时,client和server同时close对方的session.

- **server handler实现**

&emsp;&emsp;详见`DiscoveryForwardHandler`.
&emsp;&emsp;server handler设计会抽象类,提取`getForwardUrl()`和`setMessageFilter(MessageFilter messageFilter)`让子类继承实现.`MessgaeFilter`设计作为消息转换和消息过滤的接口.
```java
public interface MessageFilter {

    /**
     * convert from message,if null,not forward
     *
     * @param fromMessage from  message
     * @return forward message
     */
    Object fromMessage(AbstractWebSocketMessage fromMessage);

    /**
     * convert to message,if null,not foward
     *
     * @param toMessage to message
     * @return forward message
     */
    Object toMessage(WebSocketFrame toMessage);
}
```
&emsp;&emsp;初看这个接口很像spring的`MessageConvert`接口进行消息转换,不同的是这里设计的另一层目的是作为消息过滤,当返回null时就不会在server和client进行消息传递.
&emsp;&emsp;`getForwardUrl()`作为获取转发地址和路由的抽象接口,在`DiscoveryForwardHandler`中的实现是提取注册中心的服务地址列表进行负载均衡选取地址.如果有需求需要从数据库获取地址,重载该方法即可.

- **灵活配置handler,interceptor**

&emsp;&emsp;详见`WsHandlerRegistration`.
&emsp;&emsp;设计之初,只考虑到配置全局`AbstractWsServerHandler`和`HandshakeInterceptor`,所有websocket路由都设置为全局的处理器和拦截器.但是后来感觉确实可能有需求不同的路由需要配置不同的处理器和拦截器.所以设计了一个注册器`WsHandlerRegistration`,声明该bean并且注册多个处理器和拦截器,然后就可以配置路由时设置`handlerClass`和`interceptorClasses`属性,配置处理器和拦截器的类名,该配置会覆盖全局配置.
&emsp;&emsp;注册器的设计很简单,感觉很不优雅,但是目前没想到其他好的处理方法,因为处理器和拦截器类的构造方法可能会有参数,所以简单的使用反射无法实现.以后有思路了会进行优化.

### end

&emsp;&emsp;写这个小框架大概花了一周时间,当然一周内是边完成工作任务边写,有些地方设计的并不好,但是目前没想到好的处理方法.通过这次设计感觉自己对于spring的理解和使用还有待加强.设计模式和接口设计方面顿感书到用时方恨少,虽然以前看过很多设计模式相关的书籍,但是并没有达到灵活使用.以后会加强这方面能力的培养.
