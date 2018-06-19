---
title: SpringCloud实践之Zuul
categories: 技术
tags: [java,springcloud,网关]
date: 2017/9/27 20:00:00
---

## start

&emsp;&emsp;最近在做一个网关项目,调研之后决定使用SpringCloud体系框架,先前也做过类似的网关,不过没有使用一些开源框架,不管是服务地址注册发现,路由负载均衡转发,服务路由配置都是直接自己写的,虽然可能有些问题,但是还是有蛮多心得,后序会介绍一些相关经验.

- - -

<!--more-->

- - -

## content

### 网关

&emsp;&emsp;**何为网关?**

&emsp;&emsp;首先我这里所说的网关不是物理层面的网关,而是服务的api网关,既然是api网关,自然每个服务都需要提供api集合,每个服务需要多个节点分布式部署,所以网关层面就不是单纯的路由地址转发,还需要动态发现服务地址变更,服务负载均衡策略转发,限流熔断,可能还涉及到一些业务上的功能,比如权限认证,日志记录等,既然如此复杂,让我们zuul是怎么做的吧.

### 实践

&emsp;&emsp;因为我做的这个项目是部署在K8S上,所以并没有用到eureka作为注册中心,服务容器启动时,会自动把相关各个服务虚地址写进容器固定目录的文件下.所以,在网关层面要做的只是在项目运行时,先去解析地址文件,SpringBoot框架为我们提供了非常便捷的通过配置属性就能实例化相关配置的功能,接下来只要按照配置规范去设置属性就可以完成配置.
```java
public static void main(String[] args) {
        SpringApplication application = new SpringApplication(GatewayApplication.class);
        application.setDefaultProperties(getDefaultProperties());
        application.run(args);
    }

```

&emsp;&emsp;关于SpringCloud文档可以查看[SpringCloud中文文档](https://springcloud.cc/spring-cloud-dalston.html),关于路由的配置一般有两种,由于项目并没有使用到eureka,所以显式禁用eureka.
```yaml
ribbon:
  eureka:
    enabled: false

# first way
zuul:
  ignored-services: '*'
  routes:
    test-service:
      path: /test/**
      serviceId: test-service

test-service:
  ribbon:
    listOfServers: ${test-service.ip}:${test-service:port}

# second way
zuul:
  ignored-services: '*'
  routes:
    test-service:
      path: /test/**
      url: http://${test-service.ip}:${test-service:port}
```

&emsp;&emsp;因为项目为服务配置的是唯一的虚地址,所以看起来好像这两种配置方式都可以,实则不然,路由转发时,前一种配置会进入RibbonRoutingFilter过滤器,后一种配置会进入SimpleHostRoutingFilter过滤器.后序会介绍SpringCloud的ribbon组件,这里只需要知道ribbon是用来进行负载均衡转发即可.其实作为唯一的虚地址,并不需要负载均衡的功能,但是由于我们需要路由熔断(hystrix)的功能,而仅仅只有RibbonRoutingFilter实现了这样的功能,所以选用第一种配置方式.

&emsp;&emsp;接下来说说zuul过滤器.
![zuul过滤器图示](http://okzr61x6y.bkt.clouddn.com/zuul%E8%BF%87%E6%BB%A4%E5%99%A8%E5%9B%BE%E7%A4%BA.png)

&emsp;&emsp;zuul的主要请求生命周期包括pre,route和post等阶段.对于每个请求,都会运行具有这些类型的所有过滤器.

**ZuulServlet**
```java
 @Override
    public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();
            try {
                preRoute();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                route();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                postRoute();
            } catch (ZuulException e) {
                error(e);
                return;
            }

        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }
```

&emsp;&emsp;zuul配置的路由本质上是servlet,进入servlet的请求就会按照统一的过滤器流程走一遍,最后返回给客户端.

**FilterProcessor**
```java
 public Object runFilters(String sType) throws Throwable {
        if (RequestContext.getCurrentContext().debugRouting()) {
            Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
        }
        boolean bResult = false;
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (int i = 0; i < list.size(); i++) {
                ZuulFilter zuulFilter = list.get(i);
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
    }
```

&emsp;&emsp;如果想对zuul进行路由转发进行特殊的前处理和后处理,可以自定义自己的过滤器.过滤器直接的参数传递是通过RequestContext类进行,`RequestContext.getCurrentContext();`可以获取当前线程的RequestContext,内部其实是ThreadLocal维护的RequestContext.

**自定义pre过滤器**
```java
public class TestPreFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }
    @Override
    public int filterOrder() {
        return 1;
    }
    @Override
    public boolean shouldFilter() {
        return Boolean.TRUE;
    }
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.set(WebConstant.ZUUL_CTX_START_TIME_KEY, new Date());
        // 判断是否是登录请求
        if (!isLogin(ctx)) {
            ctx.set(WebConstant.ZUUL_CTX_ISLOGIN_KEY, Boolean.FALSE);
        } else {
            ctx.set(WebConstant.ZUUL_CTX_ISLOGIN_KEY, Boolean.TRUE);
        }
        return null;
    }
}
```

&emsp;&emsp;重点来了,关于路由的熔断(Hystrix)配置.Zuul默认使用Hystrix的信号量作熔断,至于为何不用线程池作为熔断,没有从官方找到特殊的理由,难道是因为怕线程数量太多?参考源码设置Hystrix command,在zuul的配置里确实没有配置线程池熔断的相关参数.AbstractRibbonCommand里的TODO说明后期版本里可能会对这方面进行改进.

**AbstractRibbonCommand**
```java
protected static Setter getSetter(final String commandKey,ZuulProperties zuulProperties) {

		// @formatter:off
		final HystrixCommandProperties.Setter setter = HystrixCommandProperties.Setter()
				.withExecutionIsolationStrategy(zuulProperties.getRibbonIsolationStrategy());
		if (zuulProperties.getRibbonIsolationStrategy() == ExecutionIsolationStrategy.SEMAPHORE){
			final String name = ZuulConstants.ZUUL_EUREKA + commandKey + ".semaphore.maxSemaphores";
			// we want to default to semaphore-isolation since this wraps
			// 2 others commands that are already thread isolated
			final DynamicIntProperty value = DynamicPropertyFactory.getInstance()
					.getIntProperty(name, zuulProperties.getSemaphore().getMaxSemaphores());
			setter.withExecutionIsolationSemaphoreMaxConcurrentRequests(value.get());
		} else	{
			// TODO Find out is some parameters can be set here
		}
		
		return Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("RibbonCommand"))
				.andCommandKey(HystrixCommandKey.Factory.asKey(commandKey))
				.andCommandPropertiesDefaults(setter);
		// @formatter:on
	}
```

&emsp;&emsp;那么,如果不使用信号量熔断,就是想用线程池熔断呢?一般情况下,如果仅作如下配置,那么确实会使用线程池方式熔断所有路由,但是会有一个问题,所有路由都会共享使用这个默认的线程池.
```yaml
zuul:
  ribbon-isolation-strategy: thread

hystrix:
  threadpool:
    default:
      coreSize: 20
      maxQueueSize: -1
      queueSizeRejectionThreshold: 100
  command:
    default:
      execution:
        isolation:
          strategy: THREAD
          thread:
            timeoutInMilliseconds: 20000
      circuitBreaker:
        requestVolumeThreshold: 20
        errorThresholdPercentage: 50
      metrics:
        #统计时间窗口值
        rollingStats:
          timeInMilliseconds: 60000
      requestLog:
        enabled: false
```

&emsp;&emsp;从之前AbstractRibbonCommand的`Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("RibbonCommand")).andCommandKey(HystrixCommandKey.Factory.asKey(commandKey)).andCommandPropertiesDefaults(setter);`可以看出会给所有的Command设置固定的group:`RibbonCommand`,如果使用线程池熔断,由于没有设置ThreadPoolName,会默认使用`RibbonCommand`作为ThreadPoolName,由于也没有显示配置名称为`RibbonCommand`的ThreadPool,那么就会根据默认default threadpool配置生成`RibbonCommand`,所有路由的熔断线程池就共享这个RibbonCommand ThreadPool.

&emsp;&emsp;我们的目的是要每个路由走自己的线程池,这样就不会对其他路由产生影响.幸好,Hystrix为我们留了这样的后门配置,注意`threadPoolKeyOverride: test-service`就是配置熔断的线程池名称,如果找不到该线程池就使用默认的线程池.
```yaml
hystrix:
  threadpool:
    test-service:
      coreSize: ${test-service.coreSize:40}
      maxQueueSize: ${test-service.maxQueueSize:-1}
      queueSizeRejectionThreshold: ${test-service.queueSizeRejectionThreshold:100}
  command:
    test-service:
      #thread pool name
      threadPoolKeyOverride: test-service 
      execution:
        isolation:
          strategy: THREAD
          thread:
            timeoutInMilliseconds: 50000
      circuitBreaker:
        requestVolumeThreshold: 20
        errorThresholdPercentage: 50
      metrics:
      #统计时间窗口值
        rollingStats:
          timeInMilliseconds: 60000
      requestLog:
        enabled: false
```

&emsp;&emsp;现在目的基本可以达到了,就是我们在最开始的解析路由地址路径时,显式配置hystrix的相关属性,为每一个路由配置和service-id同名的thread pool,这样就达到了每个路由只使用自己的熔断线程池了.

&emsp;&emsp;关于熔断失败处理,可以通过继承`ZuulFallbackProvider`生成通用的路由熔断失败处理响应.
```java
public class DefaultZuulFallbackProvider implements ZuulFallbackProvider {

    private static final Logger logger = LoggerFactory.getLogger(DefaultZuulFallbackProvider.class);

    private static final HttpStatus DEFAULT_FALLBACK_STATUS = HttpStatus.SERVICE_UNAVAILABLE;

    @Override
    public String getRoute() {
        return "*";
    }

    @Override
    public ClientHttpResponse fallbackResponse() {
        return new ClientHttpResponse() {
            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }

            @Override
            public InputStream getBody() throws IOException {
                RequestContext ctx = RequestContext.getCurrentContext();
                String message = ctx.get("serviceId").toString() + "服务繁忙,请稍后再试";
                String response = JSON.toJSONString(
                    new WebResponse(DEFAULT_FALLBACK_STATUS.value(), message, false, null),
                    SerializerFeature.WriteMapNullValue);
                return new ByteArrayInputStream(response.getBytes(Charsets.UTF_8));
            }

            @Override
            public HttpStatus getStatusCode() throws IOException {
                return DEFAULT_FALLBACK_STATUS;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return DEFAULT_FALLBACK_STATUS.value();
            }

            @Override
            public String getStatusText() throws IOException {
                return DEFAULT_FALLBACK_STATUS.getReasonPhrase();
            }

            @Override
            public void close() {
                RequestContext ctx = RequestContext.getCurrentContext();
                try {
                    ctx.getResponseDataStream().close();
                } catch (IOException e) {
                    logger.error("Fall back close response error, exception : {}", e);
                }
            }
        };
    }
}
```

### end

&emsp;&emsp;由于作为网关zuul组件和ribbon、hystrix整合较多,基本可以实现一个网关的功能,但总感觉少了一些什么,比如流量监控,动态路由限流等.zuul作为SpringCloud体系组件来使用似乎是唯一的网关选择,但是脱离了SpringCloud体系,我觉得一个好的网关似乎应该可以从性能和监控以及一些动态配置上做做文章,当然SpringCloud有其他的监控组件,甚至包括服务链路调用的监控,整体而言,zuul很优秀,但仍然可以变得更优秀.