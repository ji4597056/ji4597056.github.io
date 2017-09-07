---
title: Akka初探
categories: 技术
tags: [java,akka,异步]
---

## start

&emsp;&emsp;最近关于写微服务的聚合接口遇到了一些问题,由于需要将聚合接口放在统一的微服务中,可能聚合接口的量非常大,这也导致了需要分配较多的线程去支持聚合接口中的异步处理,对于调用REST接口这些远程访问,显然如果使用`CompletableFuture`,需要配置线程池相对较大的线程数才能做到支持较高的QPS和较快的响应,但是显然由于聚合接口很多,并且还进行了业务分类,为了保证线程隔离,需要配置很多线程池.更重要的问题是由于这些远程调用都是使用的`FeignClient`,`Feign`组件集成了`Hystrix`组件,由于线程隔离,也配置线程池.综上述,加上请求线程,一个http请求进来,至少需要使用到3个以上的线程数,线程数量过大,必然导致上下文切换以及线程调度等一系列性能的问题.所以,针对这些聚合接口,有必要调研一下其他解决方案.

- - -

<!--more-->

- - -

## content

### 场景

&emsp;&emsp;假设现在某一聚合接口需要异步调用远程服务A接口和远程服务B接口,并将其结果拼装后返回.为方便起见,远程服务A接口与远程服务B接口为同一服务的同一接口,并且该远程接口使用`Hystrix`熔断处理.
```java
    @HystrixCommand(fallbackMethod = "hiError")
    public String hiRemoteService(String name) {
        return defaultRestTemplate
            .getForObject("http://localhost:8080/hi?name=" + name, String.class);
    }
```

&emsp;&emsp;后序测试使用ab对聚合接口进行压测,为了保证能够看到`Hystrix`使用的线程数并且不会被熔断,需要将`Hystrix`线程数和队列数配置足够大.
```yaml
hystrix:
  threadpool:
    default:
      coreSize: 300
      queueSizeRejectionThreshold: 2000
```

&emsp;&emsp;测试方式:先预热jvm, ab压测200并发4000请求:`ab -n 4000 -c 200 url`.

### CompletableFuture

&emsp;&emsp;针对这种聚合接口调用,常见的解决方案是使用Java8的新特性`CompletableFuture`,其接口相当友好,非常适合处理相对复杂的异步、同步组合的方法聚合处理.具体使用不做过多介绍.

&emsp;&emsp;对于上述场景需要配置线程池`Executors.newFixedThreadPool(300)`,这个线程池就是文章开始所说的瓶颈所在,如果配置小了会导致响应延迟较大,QPS较低.配置大了会导致系统线程数过大.具体配置需要针对聚合接口需要达到的QPS要求以及同时异步处理的远程调用数作为依据进行配置.比如异步调用两个接口配置200,那么异步调用三个接口需要配置300才能达到大致同样的QPS.所以,聚合的接口数量越多,处理越复杂,需要配置的线程数可能也相对较多.
```java
    @GetMapping("/test/future")
    public String testFuture() throws ExecutionException, InterruptedException {
        CompletableFuture<String> future1 = CompletableFuture
            .supplyAsync(() -> helloService.hiRemoteService("Hi,"), threadPool);
        CompletableFuture<String> future2 = CompletableFuture
            .supplyAsync(() -> helloService.hiRemoteService("future"), threadPool);
        return future1.thenCombine(future2, (f1, f2) -> f1 + f2).get();
    }
```

**测试结果:**
![CompletableFuture测试](http://okzr61x6y.bkt.clouddn.com/completable%E6%B5%8B%E8%AF%95.png)
*hystrix max thread:300*

&emsp;&emsp;可以看出80%的响应相当不错,但是后序请求在阻塞队列里排队时间较长导致响应急剧下降.`Hystrix`使用的线程数达到了配置的线程数.所以,如果`Hystrix`线程数配小了,在并发较大时,会出现大量熔断.

### Quasar

&emsp;&emsp;纤程/协程这种概念对于java程序员可能相对较陌生,可以理解为轻量级的用户线程,不是由操作系统去调度,而是由应用调度去控制.这个框架似乎并不算普及流行,没有做过多研究,不发表评论.
```java
	@GetMapping("/test/quasar")
    public String testQuasar() throws ExecutionException, InterruptedException {
        return FiberUtil
            .runInFiber(() -> helloService.hiRemoteService("Hi,")
                + helloService.hiRemoteService("quasar"));
    }

    @GetMapping("/test/quasar2")
    public String testQuasar2() throws ExecutionException, InterruptedException {
        return FiberUtil.runInFiber(() -> helloService.hiRemoteService("Hi,")) + FiberUtil
            .runInFiber(() -> helloService.hiRemoteService("quasar"));
    }
```

&emsp;&emsp;`Quasar`需要java代理启动,在可能会阻塞的方法上(如远程调用)加上`@Suspendable`注解,这里测试了两种写法,在1个纤程里调用2个方法拼接结果和2个纤程各调用1个方法再拼接结果,发现测试结果几乎一致.

**测试结果:**
![Quasar测试](http://okzr61x6y.bkt.clouddn.com/quasar%E6%B5%8B%E8%AF%95.png)
*hystrix max thread:4*

&emsp;&emsp;可以看出50%的响应并没有`CompletableFuture`测试的响应快,但是整体比较稳定,即使最慢的响应也没超过1s,更大的亮点在于`Hystrix`只使用了4个线程,这让我非常诧异,通过jconsole工具发现`Quasar`在测试时一共起了5个线程,一个总的调度线程,4个fork-join线程,这就解释了为何`Hystrix`最多只使用了4个线程.

### Akka

&emsp;&emsp;`Akka`框架对于`Scala`程序员来说是必修课,但其事件驱动模型对于Java程序员而言,可能比较陌生.这里不做过多探究,因为我也只是在学习阶段,以后有机会会详细分享学习心得.`Akka`框架集成`Spring boot`需要一些特别的配置,因为`actor`实例并不是一个简单的actor引用.
```java
    @Configuration
    public class AkkaConfig {

        @Autowired
        private ApplicationContext applicationContext;

        @Bean
        public ActorSystem actorSystem() {
            ActorSystem actorSystem = ActorSystem.create("ActorSystem");
            SpringExtProvider.get(actorSystem).initialize(applicationContext);
            return actorSystem;
        }

        @Bean
        public Config akkaConfiguration() {
            return ConfigFactory.load();
        }

    }
```

```java
    public class SpringActorProducer implements IndirectActorProducer {

        final private ApplicationContext applicationContext;
        final private String actorBeanName;

        public SpringActorProducer(ApplicationContext applicationContext, String actorBeanName) {
            this.applicationContext = applicationContext;
            this.actorBeanName = actorBeanName;
        }

        @Override
        public Actor produce() {
            return (Actor) applicationContext.getBean(actorBeanName);
        }

        @Override
        public Class<? extends Actor> actorClass() {
            return (Class<? extends Actor>) applicationContext.getType(actorBeanName);
        }
    }
```

```java
    public class SpringExtension extends AbstractExtensionId<SpringExt> {

        public static final SpringExtension SpringExtProvider = new SpringExtension();

        @Override
        public SpringExt createExtension(ExtendedActorSystem system) {
            return new SpringExt();
        }

        public static class SpringExt implements Extension {

            private volatile ApplicationContext applicationContext;

            public void initialize(ApplicationContext applicationContext) {
                this.applicationContext = applicationContext;
            }

            public Props props(String actorBeanName) {
                return Props.create(SpringActorProducer.class, applicationContext, actorBeanName);
            }
        }
    }
```

&emsp;&emsp;接下来需要编写聚合接口的`actor`.
```java
    @Component("helloActor")
    @Scope("prototype")
    public class HelloActor extends AbstractActor {

        @Autowired
        private HelloService helloService;

        public static Props props() {
            return Props.create(HelloActor.class);
        }

        @Override
        public Receive createReceive() {
            return receiveBuilder()
                .match(String.class, s -> getSender().tell(helloService.hiRemoteService(s), getSelf()))
                .build();
        }
    }
```

&emsp;&emsp;作如上配置后,即可在`@component`类下自动注入`ActorSystem`,并获取`actor`引用.
```java
    @Autowired
    private ActorSystem actorSystem;

    @GetMapping("/test/akka")
    public String testAkka() throws Exception {
        Future<Object> future1 = Patterns
            .ask(actorSystem.actorOf(SpringExtProvider.get(actorSystem).props("helloActor")),
                "Hi", 10000);
        Future<Object> future2 = Patterns
            .ask(actorSystem.actorOf(SpringExtProvider.get(actorSystem).props("helloActor")),
                "Hi", 10000);
        Duration duration = Duration.create(10, TimeUnit.SECONDS);
        return Await.result(future1, duration) + (String) Await.result(future2, duration);
    }
```

**测试结果:**
![Akka测试](http://okzr61x6y.bkt.clouddn.com/akka%E6%B5%8B%E8%AF%95.png)
*hystrix max thread:12*

&emsp;&emsp;可以看出,测试结果和之前用`Quasar`的结果差不多,`Hystrix`线程数为12,想必调度线程数应该也为12.这里还没深入探究.

### end

&emsp;&emsp;这篇文章就作为简单的分析报告吧,数据虽然测过很多遍,但未必很准确.从结论而言,`CompletableFuture`确实不是很适合高并发的场景,毕竟线程资源实在很宝贵,jdk8默认线程大小是1mb,线程数较大的话会使用太多的内存,而`actor`实例的内存不过几百字节.`Quasar`框架并不是很主流,编程上确实很简单,感觉以后的java版本会出类似的协程库,毕竟用线程来做异步实在是太重了,所以并不打算深入学习`Quasar`.现在在调研`Akka`,希望能尽快熟悉这种编程方式.
