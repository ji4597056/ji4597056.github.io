---
title: SpringCloud实践之ribbon
categories: 技术
tags: [java,springcloud]
---

## start

&emsp;&emsp;今天来深入学习`spring cloud`体系下`ribbon`组件,项目中直接使用`ribbon`组件的机会不多,一般都是使用封装了`hystrix`和`ribbon`的`feign`组件来进行微服务之间的远程调用.一直没有仔细研究过`ribbon`,之前也自己写过负载均衡的http客户端,当时借鉴了`nginx`的负载策略,支持轮询、随机、加权随机、ip hash四种策略.那么`ribbon`作为`Netflix`公司开源的负载均衡客户端又有哪些亮点呢?让我们深入源码一起调研一下吧!

- - -

<!--more-->

- - -

## content

### 基本配置

- 引入maven依赖
```java
 	<dependency>
    	<groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-ribbon</artifactId>
    </dependency>
```

- 配置`RestTemplate`
```java
@Configuration
public class RibbonConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

- 使用
```java
@Service
public class RibbonService {

    @Autowired
    private RestTemplate restTemplate;
    
    public String hi(String name) {
        return restTemplate.getForObject("http://eureka-client/hi?name="+name,String.class);
    }
}
```

&emsp;&emsp;基本配置使用就那么简单,当然项目中需要引入`spring cloud`注册中心的依赖,例如`eureka`、`consul`,`RestTemplate`在发送请求时,会根据url中的`service id`从服务注册中心获取服务地址列表,然后根据负载策略选取其中一个地址发送请求.

### 源码解析

#### LoadBalanced

&emsp;&emsp;初看到这个注解,我以为是`ribbon`的做法是通过这个自定义注解在`RestTemplate`类上声明,然后通过`spring aop`机制(java原生动态代理/cglib)拦截url参数然后替换`service id`对应的服务地址.然而是我想多了.查看`LoadBalanced`代码:
```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {
}
```
&emsp;&emsp;可以看出这个注解实质上是`@Qualifier`,在`LoadBalancerAutoConfiguration`类中可以看到:
```java
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();
        
	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializer(
			final List<RestTemplateCustomizer> customizers) {
		return new SmartInitializingSingleton() {
			@Override
			public void afterSingletonsInstantiated() {
				for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
					for (RestTemplateCustomizer customizer : customizers) {
						customizer.customize(restTemplate); //定制化处理
					}
				}
			}
		};
	}
```
&emsp;&emsp;这样就可以注入`ribbon`需要特殊处理的`RestTemplate`集合,这种处理比直接使用`@Qualifier`更为优雅.在配置类`RestTemplateCustomizer`中会对`@LoadBalanced`注解的`@RestTemplate`进行定制化处理.

#### LoadBalancerInterceptorConfig

```java
	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return new RestTemplateCustomizer() {
				@Override
				public void customize(RestTemplate restTemplate) {
					List<ClientHttpRequestInterceptor> list = new ArrayList<>(
							restTemplate.getInterceptors());
					list.add(loadBalancerInterceptor);
					restTemplate.setInterceptors(list);
				}
			};
		}
	}
```
&emsp;&emsp;`RestTemplateCustomizer`主要做的事就是给`RestTemplate`增加一个`LoadBalancerInterceptor`,这应该是一个请求处理前的拦截器,具体可以参考`RestTemplate`源码,了解一下它的设计.

#### LoadBalancerInterceptor

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		// for backwards compatibility
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
}
```
&emsp;&emsp;在`LoadBalancerInterceptor`我们终于看到了拦截的流程,从`HttpRequest`中获取请求url(例如`http://eureka-client/hi`),从中解析出host(`eureka-client`),然后交给`LoadBalancerClient`处理请求.
```java

#### InterceptingRequestExecution

private class InterceptingRequestExecution implements ClientHttpRequestExecution {

		private final Iterator<ClientHttpRequestInterceptor> iterator;

		public InterceptingRequestExecution() {
			this.iterator = interceptors.iterator();
		}

		@Override
		public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
			if (this.iterator.hasNext()) {
				ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
                // 巧妙,传递自身引用,通过遍历迭代器实现处理链
				return nextInterceptor.intercept(request, body, this); 
			}
			else {
				ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), request.getMethod());
				for (Map.Entry<String, List<String>> entry : request.getHeaders().entrySet()) {
					List<String> values = entry.getValue();
					for (String value : values) {
						delegate.getHeaders().add(entry.getKey(), value);
					}
				}
				if (body.length > 0) {
					StreamUtils.copy(body, delegate.getBody());
				}
				return delegate.execute();
			}
		}
	}
```
&emsp;&emsp;这里`RestTemplate`对于拦截设计很巧妙,进入`InterceptingClientHttpRequest`类.这里对于请求拦截的处理是通过传递自身引用遍历迭代器的形式来实现责任链模式,我觉得很有新意.

#### LoadBalancerClient

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {

	// 使用从负载均衡器中挑选出来的服务实例来执行请求内容.
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

	// 使用从负载均衡器中挑选出来的服务实例来执行请求内容.
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

	// 重构url
	URI reconstructURI(ServiceInstance instance, URI original);
}

public interface ServiceInstanceChooser {

	// 选择服务实例
    ServiceInstance choose(String serviceId);
}
```
&emsp;&emsp;根据`LoadBalancerClient`的接口设计,我们大致了解了处理请求的过程为:通过`serviceId`从服务地址列表中根据策略获取`ServiceInstance`->重构原始url,将host转换成`ServiceInstance`的地址->发送请求.
&emsp;&emsp;在`RibbonLoadBalancerClient`类中可以看到`LoadBalancerClient`接口的具体实现.
```java
@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
		Server server = getServer(loadBalancer);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
				serviceId), serverIntrospector(serviceId).getMetadata(server));

		return execute(serviceId, ribbonServer, request);
	}

	@Override
	public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
		Server server = null;
		if(serviceInstance instanceof RibbonServer) {
			server = ((RibbonServer)serviceInstance).getServer();
		}
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}

		RibbonLoadBalancerContext context = this.clientFactory
				.getLoadBalancerContext(serviceId);
		RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

		try {
			T returnVal = request.apply(serviceInstance); // 向具体服务实例发送请求
			statsRecorder.recordStats(returnVal);
			return returnVal;
		}
		// catch IOException and rethrow so RestTemplate behaves correctly
		catch (IOException ex) {
			statsRecorder.recordStats(ex);
			throw ex;
		}
		catch (Exception ex) {
			statsRecorder.recordStats(ex);
			ReflectionUtils.rethrowRuntimeException(ex);
		}
		return null;
	}
```
&emsp;&emsp;`LoadBalancerClient`具体交给了`ILoadBalancer`来处理,`ILoadBalancer`通过配置`IRule`、`IPing`等接口,并向`EurekaClient`获取注册列表的信息,并默认10秒一次向`EurekaClient`发送'ping',进而检查是否更新服务列表,最后,得到注册列表后,`ILoadBalancer`根据`IRule`的策略进行负载均衡.

#### IRule
&emsp;&emsp;`IRule`是实现类很多,这里简单介绍各个策略,就不详细的贴代码,感兴趣的可以直接看源码了解这些策略的实现.

- `RandomRule`:随机选择一个服务实例.这里用的`Random`,我在自己实现随机策略时选用了`ThreadLocalRandom`,好处在于能够解决多个线程发生的竞争争夺.
- `RoundRobinRule`:轮询选择一个服务实例.这是很常用的轮询算法.
```java
	private AtomicInteger nextServerCyclicCounter;
 
	private int incrementAndGetModulo(int modulo) {
        for (;;) {
            int current = nextServerCyclicCounter.get();
            int next = (current + 1) % modulo;
            if (nextServerCyclicCounter.compareAndSet(current, next))
                return next;
        }
    }
```
- `BestAvailableRule`:选择一个最小并发请求的服务实例.
- `WeightedResponseTimeRule`:在轮询的策略基础上,过滤掉那些因为一直连接失败的被标记为circuit tripped的后端服务,并过滤掉那些高并发的的后端服务(active connections超过配置的阈值).
- `WeightedResponseTimeRule`:根据响应时间分配一个weight,相应时间越长,weight越小,被选中的可能性越低.
- `RetryRule`:对选定的负载均衡策略机上重试机制.这里用到一个`TimerTask`,到达过期时间后`Thread.currentThread().interrupt()`.
- `ZoneAvoidanceRule`:复合判断服务所在区域的性能和服务的可用性选择服务实例.

### end

&emsp;&emsp;仅仅是集成`eureka`注册中心获取服务地址列表并且负载选择服务地址,这个看似并不难的功能,通过断点调试,发现代码竟然如此复杂,远远超乎我的想象.不过确实学到了一些技巧,以后我会更加耐心地看源码和学习框架设计.