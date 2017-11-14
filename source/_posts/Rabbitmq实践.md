---
title: RabbitMq实践
categories: 技术
tags: [java,springboot,rabbitmq]
---

## start

&emsp;&emsp;以前曾使用过redis的list作为消息队列,当然redis本身是内存数据库,还是作为缓存使用才是正途,最近工作中使用到了rabbitmq作为消息队列,结合springboot详细记录下使用心得.

- - -

<!--more-->

- - -

## content

### 选择

&emsp;&emsp;现在MQ框架非常之多,比较流行的有RabbitMq、ActiveMq、ZeroMq、Kafka.在选择之前,我初步的认识是Kafka具有更好的性能优势,而RabbitMq具有更好的可靠性优势.通过网上的资料对比如下:
- **RabbitMq:**遵循`AMQP`协议,RabbitMQ的broker由`exchange`,`binding`,`queue`组成,其中`exchange`和`binding`组成了消息的路由键,客户端`Producer`通过连接channel和server进行通信,`Consumer`从`queue`获取消息进行消费(长连接,`queue`有消息会推送到`consumer`端,`consumer`循环从输入流读取数据).rabbitMQ以broker为中心:`有消息确认机制`.

- **Kafka:**遵从一般的MQ结构,`producer`,`broker`,`consumer`,以`consumer`为中心,消息的消费信息保存的客户端`consumer`上,consumer根据消费的点,从broker上批量pull数据,`无消息确认机制`,具有较高的吞吐量.

&emsp;&emsp;看似两种MQ都各有各的价值,这里贴一下网上大神的使用建议,如有说错,我不负责.

- **RbbitMQ该怎么用**
- RabbitMQ的消息应当尽可能的小,并且只用来处理实时且要高可靠性的消息.
- 消费者和生产者的能力尽量对等,否则消息堆积会严重影响RabbitMQ的性能.
- 集群部署,使用热备,保证消息的可靠性.

- - -

- **Kafka该怎么用**
- 应当有一个非常好的运维监控系统,不单单要监控Kafka本身,还要监控Zookeeper.
- 对消息顺序不依赖,且不是那么实时的系统.
- 对消息丢失并不那么敏感的系统.

### 介绍

#### 概念说明

- **`Producer`**:消息生产者.

- **`Consumer`**:消息的消费者.

- **`Queue`**:消息队列,提供了FIFO的处理机制,具有缓存消息的能力.rabbitmq中,队列消息可以设置为持久化,临时或者自动删除.设置为持久化的队列,queue中的消息会在server本地硬盘存储一份,防止系统crash,数据丢失.设置为临时队列,queue中的数据在系统重启之后就会丢失.设置为自动删除的队列,当不存在用户连接到server,队列中的数据会被自动删除.

- **`Exchange`**:类似于数据通信网络中的交换机,提供消息路由策略.rabbitmq中,producer不是通过信道直接将消息发送给queue,而是先发送给exchange.一个exchange可以和多个queue进行绑定,producer在传递消息的时候,会传递一个`ROUTING_KEY`,exchange会根据这个`ROUTING_KEY`按照特定的路由算法,将消息路由给指定的queue,和queue一样,exchange也可设置为持久化,临时或者自动删除.exchange有4种类型:`direct(默认)`,`fanout`,`topic`,和`headers`,不同类型的exchange转发消息的策略有所区别:
**Direct**
直接交换器,工作方式类似于单播,exchange会将消息发送完全匹配`ROUTING_KEY`的queue.
**Fanout**
广播是式交换器,不管消息的`ROUTING_KEY`设置什么,exchange都会将消息转发给所有绑定的queue.
**Topic**
主题交换器,工作方式类似于组播,exchange会将消息转发和`ROUTING_KEY`匹配模式相同的所有队列,比如,`ROUTING_KEY`为`user.stock`的message会转发给绑定匹配模式为`*.stock`,`user.stock*.*`和`#.user.stock.#`的队列.(`*`表是匹配一个任意词组,`#`表示匹配0个或多个词组)
**Headers**
消息体的header匹配（ignore）
**Binding**
所谓绑定就是将一个特定的exchange和一个特定的queue绑定起来.exchange和queue的绑定可以是多对多的关系.
**Virtual Host**
在rabbitmq server上可以创建多个虚拟的message broker,又叫做virtual hosts (vhosts).每一个vhost本质上是一个mini-rabbitmq server,分别管理各自的exchange和bindings.vhost相当于物理的server,可以为不同app提供边界隔离,使得应用安全的运行在不同的vhost实例上,相互之间不会干扰.producer和consumer连接rabbit server需要指定一个vhost.

#### 使用经验

&emsp;&emsp;我们使用的微服务架构体系,各个微服务做分布式部署.所以可能会遇到以下场景:

- A服务发布消息通知B服务所有节点
- A服务发布消息通知B服务单个节点
- A服务发布消息通知B服务单个(或多个)节点以及C服务单个(或多个)节点

&emsp;&emsp;上文已经说明,消息的传播路径为`producer`->`exchange`->`queue`->`consumer`.当`producer`->`exchange`时,`producer`需要指定exchange name,如果不指定,发送给默认name为`*`的`exchange`,`exchange`上可以注册多个`queue`,当`exchange`->`queue`时,根据路由规则,`exchange`会发送给若干个`queue`(0个或多个),此时会有若干个`consumer`监听`queue`,当`queue`->`consumer`时,rabbitmq会根据轮询原则发送给监听该`queue`的其中一个`consumer`.

&emsp;&emsp;rabbitmq支持生产者端和消费者端的消息确认,但是如上文所说,消息确认机制会影响吞吐量.

- **生产者**
&emsp;&emsp;当生产者将信道设置成confirm模式,一旦信道进入confirm模式,所有在该信道上面发布的消息都会被指派一个唯一的ID(从1开始),一旦消息被投递到所有匹配的队列之后,broker就会发送一个确认给生产者(包含消息的唯一ID),这就使得生产者知道消息已经正确到达目的队列了,如果消息和队列是可持久化的,那么确认消息会将消息写入磁盘之后发出,broker回传给生产者的确认消息中deliver-tag域包含了确认消息的序列号,此外broker也可以设置basic.ack的multiple域,表示到这个序列号之前的所有消息都已经得到了处理.
&emsp;&emsp;confirm模式最大的好处在于它是异步的,一旦发布一条消息,生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息,当消息最终得到确认之后,生产者应用便可以通过回调方法来处理该确认消息,如果RabbitMQ因为自身内部错误导致消息丢失,就会发送一条nack消息,生产者应用程序同样可以在回调方法中处理该nack消息.
&emsp;&emsp;在channel被设置成confirm模式之后,所有被publish的后续消息都将被confirm(即ack)或者被nack一次.但是没有对消息被confirm的快慢做任何保证,并且同一条消息不会既被confirm又被nack.

- **消费者**
&emsp;&emsp;为了保证消息从队列可靠地到达消费者,RabbitMQ提供消息确认机制(message acknowledgment).消费者在声明队列时,可以指定noAck参数,当noAck=false时,RabbitMQ会等待消费者显式发回ack信号后才从内存(和磁盘,如果是持久化消息的话)中移去消息.否则,RabbitMQ会在队列中消息被消费后立即删除它.
&emsp;&emsp;采用消息确认机制后,只要令noAck=false,消费者就有足够的时间处理消息(任务),不用担心处理消息过程中消费者进程挂掉后消息丢失的问题,因为RabbitMQ会一直持有消息直到消费者显式调用basicAck为止.
&emsp;&emsp;当noAck=false时,对于RabbitMQ服务器端而言,队列中的消息分成了两部分:一部分是等待投递给消费者的消息;一部分是已经投递给消费者,但是还没有收到消费者ack信号的消息.如果服务器端一直没有收到消费者的ack信号,并且消费此消息的消费者已经断开连接,则服务器端会安排该消息重新进入队列,等待投递给下一个消费者(也可能还是原来的那个消费者).
&emsp;&emsp;RabbitMQ不会为未ack的消息设置超时时间,它判断此消息是否需要重新投递给消费者的唯一依据是消费该消息的消费者连接是否已经断开.这么设计的原因是RabbitMQ允许消费者消费一条消息的时间可以很久很久.

### 编程实践

&emsp;&emsp;springboot集成了`spring-boot-rabbitmq`.

**配置**
```java
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5555
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
## 消费者ACK(ACK模式,默认自动ACK)
spring.rabbitmq.listener.simple.acknowledge-mode=manual
## 生产者确认(当消息被消费后回调)
spring.rabbitmq.publisher-confirms=false 
```
```java
@Configuration
public class RabbitConfig {
    public static final String RABBIT_DEFAULT_QUEUE = "default-queue";
    public static final String RABBIT_DEFAULT_EXCHANGE = "default-exchange";

    @Bean
    public Queue defaultQueue() {
    	// durable: 是否持久化
        // exclusive: 1.只对首次声明它的连接可见,其他连接不可再声明 2.会在其连接断开的时候自动删除
        // autoDelete: 在最后的消费者断开连接后自动删除
        // 当exclusive和durable同时为true时,仅exclusive为主
        return new Queue(RABBIT_DEFAULT_QUEUE, true, false, false);
    }
    
    @Bean
    public FanoutExchange defaultExchange() {
    	// durable: 是否持久化
        // autoDelete: 没有队列绑定后自动删除
        return new FanoutExchange(RABBIT_DEFAULT_EXCHANGE, true, false);
    }
    
    @Bean
    public Binding bindingExchange(Queue defaultQueue, FanoutExchange defaultExchange) {
        return BindingBuilder.bind(defaultQueue).to(defaultExchange);
    }

}
```

**生产者**
```java
 	@Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendTestExchangeMessage(String message) {
    	rabbitTemplate.convertAndSend(RabbitConfig.RABBIT_DEFAULT_EXCHANGE, RabbitConfig.RABBIT_DEFAULT_QUEUE, message);
    }
```

**消费者**
```java
@Component
@RabbitListener(queues = RabbitConfig.RABBIT_DEFAULT_QUEUE)
public class RabbitConvertReceiver {

    private static final Logger logger = LoggerFactory.getLogger(RabbitConvertReceiver.class);

    @RabbitHandler
    public void process(String message, @Header(AmqpHeaders.DELIVERY_TAG) long tag, Channel channel)) {
    	try {
        	logger.info("Receiver : " + message);
            channel.basicAck(tag, false); // 确认消息(不批量确认)
        } catch (Exception e) {
            e.printStackTrace();
            channel.basicReject(tag, true); // 拒绝消息(消息重新入队列)
        }
    }
    
}
```

**序列化方式**
&emsp;&emsp;默认使用`SerializerMessageConverter`序列化,此时消息实体类只要继承`Serializable`,即可被消费者直接接收.若想使用`json`格式进行消息发送并接收,需要主动声明`Jackson2JsonMessageConverter`,*此时需要生产者和消费者都具有相同路径下的消息实体类,否则消费者无法实例化消息实体类*.
```java
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public RabbitMessagingTemplate rabbitMessagingTemplate(RabbitTemplate rabbitTemplate,
        MessageConverter messageConverter) {
        rabbitTemplate.setMessageConverter(messageConverter);
        RabbitMessagingTemplate rabbitMessagingTemplate = new RabbitMessagingTemplate();
        rabbitMessagingTemplate.setRabbitTemplate(rabbitTemplate);
        return rabbitMessagingTemplate;
    }
```

### 其他心得

&emsp;&emsp;在遇到非java语言的生产者或消费者的情况下,建议消息使用`json`格式传递.可能出现非java生成者和java消费者,此时`Jackson2JsonMessageConverter`不可用,需要自定义`MessageConverter`,然后使用`JsonUtil`类直接解析`Message`的`body`.
```java
public class CustomMessageConvert extends Jackson2JsonMessageConverter {

    @Override
    public Object fromMessage(Message message) throws MessageConversionException {
        return message;
    }
}
```
&emsp;&emsp;在分布式场景下,可能会遇到这样一种情况,需要某服务的每个节点都订阅消息,此时需要该服务启动时动态生成队列名,用`@RabbitListener`无法传递动态队列名,此时需要声明`MessageListenerContainer`.
```java
	// 配置类中声明ListenerContainer,注入具体消息接收类
    @Bean
    public SimpleMessageListenerContainer listenerContainer(TestMessageReceiver testMessageReceiver,
        ConnectionFactory connectionFactory, MessageConverter messageConverter) throws Exception {
        String queueName = "queue_" + new Random().nextInt(1000); // 动态队列名字 
        SimpleMessageListenerContainer simpleMessageListenerContainer =
            new SimpleMessageListenerContainer(connectionFactory);
        simpleMessageListenerContainer.setQueueNames(queueName);
        simpleMessageListenerContainer.setMessageListener(testMessageReceiver);
        simpleMessageListenerContainer.setMessageConverter(messageConverter);
        // 设置手动 ACK
        simpleMessageListenerContainer.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return simpleMessageListenerContainer;
    }
```
```java
@Component
public class TestMessageReceiver extends MessageListenerAdapter {

    private static final Logger logger = LoggerFactory.getLogger(TestMessageReceiver.class);

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        try {
            logger.info("(TestMessageReceiver-process)Receiver : " + message);
            // 手动ACK
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

### end

&emsp;&emsp;原本以为很快就能写完这篇文章,由于在写的过程中一边思考工作中项目里的使用情景,并且一直测试代码验证,竟然写了一下午,应该是算是有点干货的吧.后面有时间会补上`kafka`使用心得的文章.
