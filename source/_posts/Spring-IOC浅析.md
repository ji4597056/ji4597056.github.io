---
title: Spring IOC浅析
date: 2017-02-06 10:22:55
tags:
---

BeanFactory bean工厂接口

![BeanFactory类图](http://okzr61x6y.bkt.clouddn.com/beanfactory%E7%B1%BB%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

<!--more-->

BeanDefinition bean定义接口

`
 private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();
`
资源定位->DOM解析->注册

FactoryBean

bean解析过程
- 1.转换对应beanName
这里需要注意传入的name可能不是beanName，传入的name可能是别名，也可能是FactoryBean。解析过程就是去除FactoryBean的前缀修饰符，如果是alias找到最终的beanName。

- 2.尝试从缓存中加载单例
单例是只创建一次，先从缓存中取。当然这里是尝试加载，首先尝试从缓存中取，如果失败再从singletonFactories中加载。因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，在Spring中创建bean的原则是不等bean创建完成就会创建bean的ObjectFactory提早曝光加入到缓存队列中，一旦下一个bean创建的时候需要依赖上一个bean则直接使用ObjectFactory（后面将详细讲解循环依赖的问题）

- 3.bean的实例化
如果从缓存中得到了bean的原始状态，则需要对bean进行实例化。这里有必要强调一下，缓存中记录的是原始的bean状态，并不一定是我们最终想要的bean。举个例子，假如我们需要对工厂bean进行处理，那么这里得到的其实是工厂bean的初始状态，但是我们真正需要的是工厂bean中定义的factory-method方法中返回的bean，而getObjectForBeanInstance就可以完成这个工作的。

- 4.原型模式的依赖检查
只有在单例情况下才会尝试解决循环依赖，如果存在A中有B的属性，B中有A的属性，那么就会有循环依赖的问题。

- 5.检测parentBeanFactory
从代码上看，如果缓存中没有数据的话直接转到父类工厂上去加载了，这是为何呢？条件是parentBeanFactory != null && !containsBeanDefinition(beanName)。关键在与containsBeanDefinition方法，它是在检测如果当前XML中没有配置对应的beanName，只能到parentBeanFactory中取进行一下尝试。

- 6.将存储XML配置文件的GenericBeanDefinition转换为RootBeanDefinition
因为从XML中读取Bean信息是放在GenericBeanDefinition中的，但是所有的Bean后续处理都是针对RootBeanDefinition的。这里做一下转化，如果父类bean不为空的话，则会一并合并父类的属性。

- 7.寻找依赖
因为Bean的初始化过程中，很可能会用到某些属性，而某些属性很可能是动态配置的，并且配置成依赖于其他的bean，那么这个时候就有必要先加载依赖的bean，所以在Spring的加载顺序中，在初始化某一个bean的时候首先初始化这个bean的依赖。

- 8.针对不同的scope进行bean的创建
我们知道在Spring中可以指定各种scope，默认singleton。但是还有些其他的配置诸如prototype、Request之类的。在这个步骤中，Spring会根据不同的配置进行不同的初始化策略。

- 9.类型转换
程序到这里返回bean基本结束了，通常对该方法的调用参数requiredType是为空的，但是可能会存在这种情况，返回的bean其实是个String，但是requiredType却传入Integer类型，那么这个时候这个步骤就起作用了，它的功能是将返回的bean转换为requiredType所指定的类型。当然，Spring转换为Integer是最简单的一种转换，在Spring中提供了各种各样的转换器，用户也可以自己扩展转换器来满足需求。