---
title: spring循环依赖解析
categories: 技术
tags: [java,spring]
---

## start

&emsp;&emsp;写这边文章的初衷是因为面试的时候遇到这个问题没有很好地答出来.以前只是粗略地看过部分源码,spring研究的不够深入.不过后来想了想,不应该产生面试驱动学习的心态,作为纯粹的热爱技术的程序员,不管这是不是面试的热点,不以结果为导向,不断探寻最佳解决方案的过程才是应有的技术素养.

- - -

<!--more-->

- - -

## content

### 问题

&emsp;&emsp;这里先抛出两个问题,也是面试时面试官抛给我的问题：

- 怎么解决两个bean相互依赖注入的问题？
- 各种循环依赖注入的情况都可以解决吗？

&emsp;&emsp;最终我们要找到问题的答案,但先不急着看源码,先写一些测试类.(往往直接看源码可能事倍功半,因为没有基本的认识和思考,很难在源码中学到设计之精华)

```java
	 ApplicationContext context;
	 
	 private void setContext(Class config) {
        context = new AnnotationConfigApplicationContext(config);
    }
```

1. `ABean`依赖`BBean`,`BBean`依赖`ABean`(非构造函数参数注入)
```java
	// ABean
    @Component
    public static class ABean {
        @Autowired
        private BBean bBean;

        public BBean getbBean() {
            return bBean;
        }
    }

    // BBean
    @Component
    public static class BBean {
        @Autowired
        private ABean aBean;

        public ABean getaBean() {
            return aBean;
        }
    }
	
	@Configuration
    @Import({ABean.class, BBean.class})
    public static class Config1 {

    }

    @Test
    public void test1() {
		// 没有出现循环依赖报错
        setContext(Config1.class);
        Assert.assertNotNull(context.getBean(ABean.class).getbBean()); //OK
        Assert.assertNotNull(context.getBean(BBean.class).getaBean()); //OK
    }
```

2. `CBean`依赖`DBean`,`DBean`依赖`EBean`,`EBean`依赖`CBean`(非构造函数参数注入)
```java
	// CBean
    @Component
    public static class CBean {
        @Autowired
        private DBean dBean;

        public DBean getdBean() {
            return dBean;
        }
    }

    // DBean
    @Component
    public static class DBean {
        @Autowired
        private EBean eBean;

        public EBean geteBean() {
            return eBean;
        }
    }

    // EBean
    @Component
    public static class EBean {
        @Autowired
        private CBean cBean;

        public CBean getcBean() {
            return cBean;
        }
    }
	
	@Configuration
    @Import({CBean.class, DBean.class, EBean.class})
    public static class Config2 {

    }

    @Test
    public void test2() {
		// 没有出现循环依赖报错
        setContext(Config2.class);
        Assert.assertNotNull(context.getBean(CBean.class).getdBean()); //OK
        Assert.assertNotNull(context.getBean(DBean.class).geteBean()); //OK
        Assert.assertNotNull(context.getBean(EBean.class).getcBean()); //OK
    }
```

3. `FBean`依赖`GBean`,`GBean`依赖`FBean`(构造函数参数注入)
```java
    // FBean
    @Component
    public static class FBean {
        private GBean gBean;

        @Autowired
        public FBean(GBean gBean) {
            this.gBean = gBean;
        }

        public GBean getgBean() {
            return gBean;
        }
    }

    // GBean
    @Component
    public static class GBean {
        private FBean fBean;

        @Autowired
        public GBean(FBean fBean) {
            this.fBean = fBean;
        }

        public FBean getfBean() {
            return fBean;
        }
    }
	
	@Configuration
    @Import({FBean.class, GBean.class})
    public static class Config4 {

    }

    @Test
    public void test4() {
		// 出现循环依赖报错(Is there an unresolvable circular reference?)
        boolean exception = false;
        try {
            setContext(Config4.class);
            context.getBean(FBean.class).getgBean();
            context.getBean(GBean.class).getfBean();
        } catch (Exception e) {
            e.printStackTrace();
            exception = true;
            Assert.assertNotNull(e);
        }
        Assert.assertTrue(exception); // OK
    }
```

4. `HBean`依赖`ABean`、`BBean`(构造函数参数注入)
```java
    // HBean
    @Component
    public static class HBean {
        private ABean aBean;

        private BBean bBean;

        @Autowired
        public HBean(ABean aBean, BBean bBean) {
            this.aBean = aBean;
            this.bBean = bBean;
        }

        public ABean getaBean() {
            return aBean;
        }

        public BBean getbBean() {
            return bBean;
        }
    }
	
	@Configuration
    @Import({ABean.class, BBean.class, HBean.class})
    public static class Config3 {

    }

    @Test
    public void test3() {
		// 没有出现循环依赖报错
        setContext(Config3.class);
        Assert.assertNotNull(context.getBean(ABean.class).getbBean()); // OK
        Assert.assertNotNull(context.getBean(BBean.class).getaBean()); // OK
        Assert.assertNotNull(context.getBean(HBean.class).getaBean()); // OK
        Assert.assertNotNull(context.getBean(HBean.class).getbBean()); // OK
    }
```

5. `IBean`依赖`JBean`,`JBean`依赖`IBean`(非构造函数参数注入,两者皆为`prototype`类型)
```java
    // IBean
    @Component
    @Scope("prototype")
    public static class IBean {
        @Autowired
        private JBean jBean;

        public JBean getjBean() {
            return jBean;
        }
    }

    // JBean
    @Component
    @Scope("prototype")
    public static class JBean {
        @Autowired
        private IBean iBean;

        public IBean getiBean() {
            return iBean;
        }
    }
	
	@Configuration
    @Import({IBean.class, JBean.class})
    public static class Config5 {

    }

    @Test
    public void test5() {
		// 出现循环依赖报错(Is there an unresolvable circular reference?)
        boolean exception = false;
        try {
            setContext(Config5.class);
            context.getBean(IBean.class).getjBean();
            context.getBean(JBean.class).getiBean();
        } catch (Exception e) {
            e.printStackTrace();
            exception = true;
            Assert.assertNotNull(e);
        }
        Assert.assertTrue(exception); // OK
    }
```

6. `KBean`依赖`LBean`,`LBean`依赖`KBean`(非构造函数参数注入,其中一个为`prototype`类型)
```java
    // KBean
    @Component
    @Scope("prototype")
    public static class KBean {
        @Autowired
        private LBean lBean;

        public LBean getlBean() {
            return lBean;
        }
    }

    // LBean
    @Component
    public static class LBean {
        @Autowired
        private KBean kBean;

        public KBean getkBean() {
            return kBean;
        }
    }
	
	@Configuration
    @Import({KBean.class, LBean.class})
    public static class Config6 {

    }

    @Test
    public void test6() {
	   // 没有出现循环依赖报错
       setContext(Config6.class);
       Assert.assertNotNull(context.getBean(KBean.class).getlBean()); // OK
       Assert.assertNotNull(context.getBean(LBean.class).getkBean()); // OK
    }
```

&emsp;&emsp;通过这几个测试,可以得出假设性的结论:

- 属性注入(或`setter`方式注入)不会产生循环依赖 
- 构造器参数注入会产生循环依赖
- `prototype`类型的`bean`互相注入会产生循环依赖,但`prototype`类型的`bean`和`singleton`类型的`bean`之间互相注入不会产生循环依赖

### 源码解析

&emsp;&emsp;虽然得出了假设性的结论,但我们不能止步于此,因为还需要知道结论是否正确以及产生这样结论的原因,为了寻求真相,我们需要在源码中找出信服的答案.(想要完全说服别人,首先应该完全说服自己)

&emsp;&emsp;spring源码还是比较复杂的,既然本文的重点是循环依赖,那么其他相关的代码不做深入解读.

&emsp;&emsp;先来看一下spring容器(`AbstractApplicationContext`)的初始化过程,首先进行`BeanDefinition`(将用户定义好的`bean`表示成IOC容器内部的数据结构,即`BeanDefinition`)的资源定位并解析(比如`FileSystemXmlApplicationContext`通过XML文件解析成`BeanDefinition`,又比如测试中用到的`AnnotationConfigApplicationContext`通过注解解析成`BeanDefinition`),此时`bean`定义载入成功后并不代表已经完成了依赖注入,只有在第一次通过`getBean`向容器索取`bean`的时候才会发生依赖注入.其后,IOC容器初始化通过一个重要模板方法`refresh()`完成.(不是本文重点,这里不深入各个方法详细说明)

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			// 若已初始化,则销毁之前创建的BeanFactory以及所有的单例Bean,重新构造DefaultListableBeanFactor,并加载BeanDefinition
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// 设置BeanFactory一些必备的属性及配置
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 设置BeanFactory的后置处理
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				// 调用BeanFactory的后处理器,这些后处理器是在Bean定义中向容器注册的
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 注册Bean的后处理器,在Bean创建过程中调用
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				// 对上下文中的消息源进行初始化
				initMessageSource();

				// Initialize event multicaster for this context.
				// 初始化上下文中的事件机制
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 初始化其他的特殊Bean
				onRefresh();

				// Check for listener beans and register them.
				// 检查监听Bean并且将这些Bean向容器注册
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 实例化所有非懒加载(not-lazy-init)Bean(默认为非懒加载)
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 发布容器事件,结束Refresh过程
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				// 若发生异常,则销毁已经在前面过程中生成的单例Bean
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

&emsp;&emsp;既然依赖注入发生在`getBean`,下面重点解读这个过程,实际取得`bean`并触发依赖注入发生的方法为`doGetBean`.

```java
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		
		// 返回Bean名称,如果是factoryBean则删除前缀'&',并将别名解析为规范名称.
		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		// 从缓存中取单例模式的Bean(详情见下文getSingleton())
		Object sharedInstance = getSingleton(beanName);
		// 单例Bean存在的情况下,那么beanName返回的肯定是单例类,但是这里还需要判断是不是FactoryBean
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			// 如果sharedInstance是FactoryBean,通过FactoryBean的getObject()获取Bean
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		// 单例Bean不存在的情况下,此时有可能beanName是单例,但之前并没有实例化,或者是Prototype类型
		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
        	// 首先判断是否为循环依赖,这里的循环依赖指的是Prototype类型
			// 如果在prototypesCurrentlyInCreation(此为hashmap,下文会看到在创建Prototype类型的Bean时会将beanName放入该map中,结束时会从map中移除)中找到该beanName,则报循环依赖异常
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			// 这里对IOC容器中的BeanDefinition是否存在进行检查,检查是否能在当前的BeanFactory中获取需要的Bean.
			// 如果当前的工程中找不到,则尝试到双亲BeanFactory中去取,如果当前的双亲工厂中取不到,那就顺着双亲工厂链一直向上查找
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
			
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				// 确保该Bean所依赖Bean都已经生成,即调用getBean(dep)
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						getBean(dep);
					}
				}

				// Create bean instance.
				// 开始创建Bean实例
				// 1.Singleton类型的Bean,则生成单例对象
				if (mbd.isSingleton()) {
					// 这里是解决单例循环依赖的关键,见下文getSingleton()和doCreateBean()(createBean()通过调用doCreateBean()生成Bean)
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					// 如果sharedInstance是FactoryBean,通过FactoryBean的getObject()获取bean
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				
				// 2.Prototype类型的Bean,需要创建一个新对象
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						// 将beanName放入prototypesCurrentlyInCreation中,用以判断循环依赖
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						// 将beanName从prototypesCurrentlyInCreation中移除
						afterPrototypeCreation(beanName);
					}
					// 如果prototypeInstance是FactoryBean,通过FactoryBean的getObject()获取bean
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
				
				// 3.处理其他scope的Bean
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		// 这里对创建的Bean进行类型检查,如果没有问题就返回这个新创建的Bean,此时这个Bean已经是包含了依赖关系的Bean
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

```java
	// 缓存创建的单例对象:bean名字 --> bean对象
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

	// 缓存单例的factory,就是ObjectFactory,bean name --> ObjectFactory
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

	// 缓存创建的单例对象,功能和singletonObjects不一样,在bean构造成功之后,属性初始化之前会把对象放入到这里,主要是用于解决属性注入的循环引用:bean name --> bean instance 
	private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

	// 记录在创建单例对象中循环依赖的问题,还记得Prototype中又记录创建过程中依赖的map吗?在Prototype中只要出现了循环依赖就抛出异常,而在单例中会尝试解决
	private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));
```

```java
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// 这里尝试从三个缓存中获取单例Bean
		// singletonObjects -> earlySingletonObjects -> singletonFactory
		Object singletonObject = this.singletonObjects.get(beanName);
		// 先从singletonObjects中找,singletonObjects没有且bean正在创建过程中(判断beanName是否在singletonsCurrentlyInCreation中)则继续查询
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				// 再从earlySingletonObjects中找
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					// 最后从singletonFactories中找,若找到则将其升级到earlySingletonObjects中
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
			// 从singletonObjects中查询Bean,未查到则创建
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				// 将beanName放入singletonsCurrentlyInCreation中,若已有则循环依赖报错
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					// 调用参数singletonFactory获取singletonObject
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					// 将beanName从singletonsCurrentlyInCreation移除
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					// 主要进行以下步骤,将缓存升级为singletonObjects
					// this.singletonObjects.put(beanName, singletonObject);
					// this.singletonFactories.remove(beanName);
					// this.earlySingletonObjects.remove(beanName);
					// this.registeredSingletons.add(beanName);
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		// BeanWrapper(BeanWrapper是对Bean的包装)用来持有创建出来的Bean对象
		BeanWrapper instanceWrapper = null;
		// 如果是Singleton类型,先把缓存中的同名Bean清除
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			// 如果没有实例化则创建新的BeanWrapper,见下文createBeanInstance()
			// 若构造函数注入,需要调用getBean(),注意此时三级缓存中并没有该Bean,会因为循环依赖报错
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			// 此时构造成功,但属性还没注入的,加入singletonFactories缓存
			// this.singletonFactories.put(beanName, singletonFactory);
			// this.earlySingletonObjects.remove(beanName);
			// this.registeredSingletons.add(beanName);
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			// 这里是对Bean的初始化,依赖注入便发生在这里,由于代码嵌套太深就不贴代码了
			// 简单说明下,其中是通过AutowiredAnnotationBeanPostProcessor这个BeanPostProcessor进行属性注入,即对依赖的Bean调用getBean()获取后注入
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			// 将beanName加入earlySingletonObjects缓存
			// 与singletonObjects不同的是当一个单例Bean被放到里面后,那么在Bean在创建过程中,就可以通过 getBean()方法获取到,可以用来检测循环引用
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

```java
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		// 确认需要创建的Bean实例的类可以实例化
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		if (mbd.getFactoryMethodName() != null)  {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}

		// Need to determine the constructor...
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
			// 使用构造函数对Bean进行实例化
			// 如果是通过构造器注入,可能会出现循环依赖的问题,由于代码嵌套太深就不贴代码了
        	// 假设在A初始化的时候发现构造函数依赖B,就会去实例化B然后B也会运行到这段逻辑,构造函数中发现依赖A,这个时候就会抛出循环依赖的异常
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// No special handling: simply use no-arg constructor.
		// 使用默认的构造函数对Bean进行实例化
		return instantiateBean(beanName, mbd);
	}
```

&emsp;&emsp;代码贴了那么多,现在简单说明下:

&emsp;&emsp;解决单例Bean直接循环依赖(非构造器注入)的核心为Bean的三级缓存(`singletonObjects` -> `earlySingletonObjects` -> `singletonFactory`).假设A依赖B,B依赖A(A、B均为单例).

- 首先调用`getBean(A)`,此时三级缓存都未命中A,将创建A的`FactoryBean`放入`singletonFactory`缓存中,然后进行属性注入.(此时第一次`getBean(A)`的流程并未结束)
- 由于A依赖B则调用`getBean(B)`,此时三级缓存都未命中B,将创建B的`FactoryBean`放入`singletonFactory`缓存中,然后进行属性注入.(此时第一次`getBean(B)`的流程并未结束)
- 由于B依赖A则调用`getBean(A)`(这是第二次调用),此时在`singletonFactory`缓存中命中A,并将A从`singletonFactory`缓存移入`earlySingletonObjects`缓存,即得到未完全依赖注入的A,因为已从缓存中取到A,不会再进行属性注入,此时不再调用`getBean(B)`,如果一直这样A->B->A->B的调用下去就是死循环了.(此时第二次`getBean(A)`的流程结束)
- 回到第一次`getBean(B)`的过程中,将未完全依赖注入的A注入到B的属性中,将B从`singletonFactory`缓存移入`earlySingletonObjects`缓存,将B从`earlySingletonObjects`缓存移入到`singletonObjects`缓存,此时即生成了依赖注入完全的B,以后获取B直接可从`singletonObjects`缓存中获取.(此时第一次`getBean(B)`的流程结束)
- 回到第一次`getBean(A)`的过程中,将已完全依赖注入的B注入到A的属性中,由于A已在`earlySingletonObjects`缓存中,将A从`earlySingletonObjects`缓存移入到`singletonObjects`缓存,此时即生成了依赖注入完全的A,以后获取A直接可从`singletonObjects`缓存中获取.(此时第一次`getBean(A)`的流程结束)

&emsp;&emsp;如果A、B均为`Prototype`类型,并没有上述那一套三级缓存机制解决循环依赖,而是直接报错.如果A、B中一个为`Singleton`类型,一个为`Prototype`类型并不会产生循环依赖.

&emsp;&emsp;如果A、B为构造器循环依赖注入(单例),会直接报错.因为A需要获取构造函数中参数B而调用`getBean(B)`,从而触发第二次调用`getBean(A)`,此时三级缓存中并没有A但A在`singletonsCurrentlyInCreation`中则报错,见源码中`doCreateBean()`方法和`getSingleton()`方法.

&emsp;&emsp;那么涉及到这种构造器注入而导致报错的问题该如何解决呢?见如下测试:

```java
    // MBean
    @Component
    public static class MBean {

        private NBean nBean;

        @Autowired
        @Lazy
        public MBean(NBean nBean) {
            this.nBean = nBean;
        }

        public NBean getnBean() {
            return nBean;
        }
    }

    // NBean
    @Component
    public static class NBean {

        private MBean mBean;

        @Autowired
        @Lazy
        public NBean(MBean mBean) {
            this.mBean = mBean;
        }

        public MBean getmBean() {
            return mBean;
        }
    }
	
	@Configuration
    @Import({MBean.class, NBean.class})
    public static class Config7 {

    }

    @Test
    public void test7() {
        setContext(Config7.class);
        Assert.assertNotNull(context.getBean(MBean.class).getnBean()); // OK
        Assert.assertNotNull(context.getBean(NBean.class).getmBean()); // OK
		Assert.assertNotSame(context.getBean(MBean.class).getnBean(), context.getBean(NBean.class)); // OK
    }
```

&emsp;&emsp;从测试中可以看到只是通过增加`@Lazy`注解便不再报错,并且很有意思的现象是`context.getBean(MBean.class).getnBean() != context.getBean(NBean.class)`.因为这种情况的处理逻辑并不一样(之前的逻辑是`getBean(A)`时由于A依赖B,所以在这个过程中主动触发`getBean(B)`),见如下获取依赖的代码(类`ContextAnnotationAutowireCandidateResolver`).

```java
	@Override
	@Nullable
	public Object getLazyResolutionProxyIfNecessary(DependencyDescriptor descriptor, @Nullable String beanName) {
		// 如果是延迟加载,则调用buildLazyResolutionProxy()获取目标依赖的代理对象作为依赖注入
		return (isLazy(descriptor) ? buildLazyResolutionProxy(descriptor, beanName) : null);
	}
	
	protected Object buildLazyResolutionProxy(final DependencyDescriptor descriptor, final @Nullable String beanName) {
		Assert.state(getBeanFactory() instanceof DefaultListableBeanFactory,
				"BeanFactory needs to be a DefaultListableBeanFactory");
		final DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) getBeanFactory();
		TargetSource ts = new TargetSource() {
			@Override
			public Class<?> getTargetClass() {
				return descriptor.getDependencyType();
			}
			@Override
			public boolean isStatic() {
				return false;
			}
			@Override
			public Object getTarget() {
				// 只有真正调用依赖对象的方法或属性的时候才会通过代理获取真正的目标依赖对象,此时才会触发对目标依赖对象getBean()
				Object target = beanFactory.doResolveDependency(descriptor, beanName, null, null);
				if (target == null) {
					Class<?> type = getTargetClass();
					if (Map.class == type) {
						return Collections.EMPTY_MAP;
					}
					else if (List.class == type) {
						return Collections.EMPTY_LIST;
					}
					else if (Set.class == type || Collection.class == type) {
						return Collections.EMPTY_SET;
					}
					throw new NoSuchBeanDefinitionException(descriptor.getResolvableType(),
							"Optional dependency not present for lazy injection point");
				}
				return target;
			}
			@Override
			public void releaseTarget(Object target) {
			}
		};
		ProxyFactory pf = new ProxyFactory();
		pf.setTargetSource(ts);
		Class<?> dependencyType = descriptor.getDependencyType();
		if (dependencyType.isInterface()) {
			pf.addInterface(dependencyType);
		}
		return pf.getProxy(beanFactory.getBeanClassLoader());
	}
```

&emsp;&emsp;可以在源码中看到对于`@Lazy`的构造器注入的情况下,会注入一个目标依赖的代理对象(通过`cglib`动态代理),只有真正触发对目标依赖的属性或方法调用时才通过`getBean()`去获取目标依赖,此时目标依赖已在缓存(`singletonObjects`)中.这里我产生一个想法,使用了`@Lazy`构造器注入,但是在构造器中主动调动依赖对象会如何呢?理论上应该循环依赖报错,经测试果然如此.

### end

&emsp;&emsp;这篇文章写得还算认真,至少还是能说服我自己的.机会是留给有准备的人,只有日常技术积累足够,才能在面试中给面试官留下深刻的影响.最后想说一下,spring源码确实曾经看过,但当时浅尝辄止,很多逻辑其实并不清晰,通过这次无数次断点,带着问题和思考去看源码,终于解答了很多疑惑.
