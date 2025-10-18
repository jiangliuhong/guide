---
title: Spring核心原理
categories: 
    - Java
    - Spring
date: 2018-10-18 21:52:08
tags:
	- Spring
cover: https://static.jiangliuhong.top/blogimg/spring/spring.png   
---
# Spring核心原理

> 在Spring中拥有许多的组件，但核心部分主要为：Beans、Core、Context、Expression，其中最为主要的为Core、与Beans，它们提供了最为核心的IOC和依赖注入功能。下文主要从这两个着手进行说明。

## 设计思想

Spring5架构图：

![spring5](https://static.jiangliuhong.top/blogimg/spring/spring5.png)

Spring框架设计理念

在Spring框架中，其最核心组件应属Beans，Spring-Beans模块是所有应用都必须使用的，它包含了访问配置文件、创建和管理Bean以及进行控制反转(`IOC`，Inversion of Control)、依赖注入(`DI`,Dependency Injection)操作相关的所有类。

在IBM developerWorks一文中([https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/index.html](https://www.ibm.com/developerworks/cn/java/j-lo-spring-principle/index.html))这样说道：Spring就是面向Bean编程（BOP,Bean Oriented Programming），Bean 在 Spring 中才是真正的主角。



## Bean生命周期

> 在Spring中Bean贯穿整个Spring应用，其生命周期应是从Spring容器创建后开始，直至Spring容器主动或被动销毁Bean。

在Spring中，Bean默认为单例模式，即singleton属性默认为false，从BeanFactory中取得的Bean实例为在其初始化就产生一个新的对象，而不是每次获取的时候都产出一个新的对象。当然我们也可以在初始化Bean的时候设置其singleton属性为true，使这个Bean变成多例模式，在getBean的时候，Spring都会产出一个新的对象，类似于Java的中的new Object操作。但设置其为多例，应避免多线程同时存取共享资源所引发的数据不同步问题。

然后在Spring中，一个Bean从创建到销毁，大致需要经历一下几个步骤（其具体的实现方式，将在下放继续阐述）：

1. 实例化Bean，根据Spring配置，执行包扫描操作，并对其进行实例化操作。
2. 根据Spring上线文对实例化的Bean进行配置，其中包括依赖注入。
3. 判断是否实现了BeanNameAware接口，如果有会执行setBeanName方法去设置这个Bean的id，即自己设置Bean在BeanFactory中的名字。
4. 判断是否实现了BeanFactoryAware接口，如果有则执行setBeanFactory方法，使得Bean可以获取自己的工厂，从而可以使用工厂的getBean方法。
5. 判断是否实现了ApplicationContextAware接口，如果有则执行setApplicationContext方法，使得在Bean中可以获取Spring上下文，从而可以获取通过上下文去getBean，该步骤与上一步作用大致相同，但通过该中方式能实现的功能却更加丰富。
6. 判断是否实现了BeanPostProcessor接口，如果有则调用postProcessBeforeInitialization方法，这一步属于实例化Bean的前置操作，在经过该步骤后，即Bean实例化后同样也会执行一个操作，详情见第八条。
7. 判断Bean是否配置了init-method，如果有，在Bean初始化之后会默认执行一次init-method指定的方法。
8. 判断Bean是否实现了BeanPostProcessor接口，如果有则调用postProcessAfterInitialization方法，该方法属于Bean实例化后的操作。在经过这个步骤后，Bean的初始化操作就完成了。
9. 当Bean实例化完成后，当Bean不再被需要的时候会执行销毁操作。一般是在ApplicationContext执行close方法时会进行销毁操作。
10. 在销毁过程中，判断是否实现了DisposableBean接口，如果有则执行destroy方法。
11. 判断Bean是否配置了destroy-method，如果有，在销毁过程中会默认执行一次destroy-method方法。

## Bean初始化过程

在上一篇关于‘Spring初始化过程’的文章最后写到Spring初始化过程中最后执行的核心方法是AbstractRefreshableApplicationContext类的refresh方法。这里再次将其源码贴出.

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
               // 1.为应用上下文的刷新做准备--设置时间、记录刷新日志、初始化属性源中的占位符(事实上什么都没做)和验证必要的属性等
			// Prepare this context for refreshing.
			prepareRefresh();
			// 2.让子类刷新内部的bean factory
			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
            //3.为上下文准备bean factory
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);
			try {
                 // 4.bean factory 后置处理
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);
				// 5.调用应用上下文中作为bean注册的工厂处理器
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);
				// 6.注册拦截创建bean的bean处理器
				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);
				 // 7.初始化消息源
				// Initialize message source for this context.
				initMessageSource();
				 // 8.初始化事件广播
				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();
				// 9.初始化特定上下文子类中的其它bean
				// Initialize other special beans in specific context subclasses.
				onRefresh();
				 // 10.注册监听器bean
				// Check for listener beans and register them.
				registerListeners();
				// 11.实例化所有的单例bean
				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);
 				// 12.发布相应的事件
				// Last step: publish corresponding event.
				finishRefresh();
			}catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}
				 //销毁错误的资源
				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();
				//重置刷新标志
				// Reset 'active' flag.
				cancelRefresh(ex);
				//主动抛出异常
				// Propagate exception to caller.
				throw ex;
			}
			finally {
                //重置内存缓存
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

通过源码可以看出该方法是构建整个IOC容器的完成过程。其中每一行代码都是创建容器的一个流程。其中主要包括以下几个步骤：

1. 构建BeanFactory
2. 添加事件处理
3. 创建 Bean 实例对象并构建Bean关系
4. 触发被监听的事件 

### 构建BeanFactory

构建BeanFactory的操作主要包括步骤1、2、3。其中第一步在我看来并未做啥重要的事情，我们只需将焦点定在二三步即可。

在第二步中，obtainFreshBeanFactory方法主要调用了AbstractRefreshableApplicationContext#refreshBeanFactory方法：

```java
protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

在上述代码中清晰可见的说明了BeanFactory的创建过程，首先判断当前容器是否存在BeanFactory，如果有则销毁后在进行创建。通过try/catch中代码可见，BeanFactory的是DefaultListableBeanFactory的实例对象。

通过Idea查看DefaultListableBeanFactory的类图如下：

![DefaultListableBeanFactory.png](https://static.jiangliuhong.top/blogimg/spring/DefaultListableBeanFactory.png)

从这个图中发现除了 BeanFactory 相关的类外，还发现了与 Bean 的 register 相关。这在 refreshBeanFactory 方法中有一行 loadBeanDefinitions(beanFactory) 将找到答案，这个方法将开始加载、解析 Bean 的定义，也就是把用户定义的数据结构转化为 Ioc 容器中的特定数据结构。 

对于WEB应用而言，其调用的是XmlWebApplicationContext#loadBeanDefinitions方法。

在BeanFactory创建成功后，Spring将创建的对象交于第三步，即prepareBeanFactory方法为其添加一些 Spring 本身需要的一些工具类。

### 添加事件处理

在第4、5、6步中，这三行代码对 Spring 的功能扩展性起了至关重要的作用 。其中第4、5步的代码主要是让你可以在Bean已经被创建，但未被初始化之前对已经构建的 BeanFactory 的配置做修改；第6步代码主要是让你可以对以后再创建 Bean 的实例对象时添加一些自定义的操作 。

第4步，即postProcessBeanFactory方法。主要功能为允许上下文能对BeanFactory做一些处理。比如：AbstractRefreshableWebApplicationContext抽象类实现了该方法，并在方法中对Servlet做了一些处理。

第5步，即invokeBeanFactoryPostProcessors方法主要是获取实现 BeanFactoryPostProcessor 接口的子类。其中主要包括执行BeanDefinitionRegistryPostProcessor类型的postProcessBeanDefinitionRegistry方法，以及执行非BeanDefinitionRegistryPostProcessor类型的postProcessBeanFactory方法。当然，该方法传入的参数是ConfigurableListableBeanFactory 类型，我们仅能对BeanFactory的一些配置做修改。

第6步，即registerBeanPostProcessors 方法也是可以获取用户定义的实现了 BeanPostProcessor 接口的子类，并执行把它们注册到 BeanFactory 对象中的 beanPostProcessors 变量中。BeanPostProcessor 中声明了两个方法：postProcessBeforeInitialization、postProcessAfterInitialization 分别用于在 Bean 对象初始化时执行。可以执行用户自定义的操作。 

第7、8、9、10步的方法主要是初始化监听事件和对系统的其他监听者的注册，监听者必须是 ApplicationListener 的子类。 在容器启动时，Spring会调用ApplicationStartListener的onApplicationEvent方法。

### 创建 Bean 实例对象并构建Bean关系

Bean的实例化过程是第11步开始的，即finishBeanFactoryInitialization方法，而在finishBeanFactoryInitialization方法中核心方法又为preInstantiateSingletons（DefaultListableBeanFactory类）。在该方法中，首先拿到所有beanName，然后在实例化的时候会判断bean是否为FactoryBean，顾名思义，这是一个特殊的工厂Bean，可以产生Bean的Bean。

这里的产生 Bean 是指 Bean 的实例，如果一个类继承 FactoryBean 用户只要实现他的 getObject 方法，就可以自己定义产生实例对象的方法。然而在 Spring 内部这个 Bean 的实例对象是 FactoryBean，通过调用这个对象的 getObject 方法就能获取用户自定义产生的对象，从而为 Spring 提供了很好的扩展性。Spring 获取 FactoryBean 本身的对象是在前面加上 & 来完成的。 

在Bean的实例化主要分两个步骤：

- Bean不为抽象、单例、非懒加载。
  - 判断Bean是否为FactoryBean，是则执行FactoryBean相关的操作，否则直接调用getBean方法产生实例。
- 在singleton的bean初始化完了之后调用SmartInitializingSingleton的afterSingletonsInstantiated方法。

通过跟进Bean实例化代码可以发现getBean方法最后指向的是AbstractBeanFactory类的抽象方法createBean，其实现类为AbstractAutowireCapableBeanFactory。在该方法中有一个步骤会去查找Bean的依赖关系，并对其进行依赖注入操作。这里画一个时序图来作说明。

![实例化Bean时序图](https://static.jiangliuhong.top/blogimg/spring/%E5%AE%9E%E4%BE%8B%E5%8C%96Bean%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

需要注意的是，在resolveValueIfNecessary方法中，不仅有调用resolveReference，同样的还有resolveInnerBean，即解析内部的Bean引用。

### 触发被监听的事件 

在经历第11步后，上下文的创建就已经基本完成了，这时Spring会执行finishRefresh方法，完成此上下文的刷新。其中包括LifecycleProcessor的onRefresh方法，并执行ContextRefreshedEvent事件。例如：执行SpringMVC的事件。

## 总结

在Bean声明周期中曾讲到，Spring在初始化Bean的时候先后对调用BeanPostProcessor接口的postProcessBeforeInitialization、postProcessAfterInitialization方法。利用这一特性我们可以在Bean中实现BeanPostProcessor接口，然后再方法体中加入自己的逻辑。

对于Spring的IOC容器而言，除了BeanPostProcessor，还有BeanFactoryPostProcessor。顾明思议，BeanFactoryPostProcessor是在构建BeanFactory和构建Bean对象时调用。IOC容器允许BeanFactoryPostProcessor在容器初始化任何Bean之前对BeanFactory配置进行修改。

在Spring的IOC容器中还有一个特殊的Bean，即FactoryBean，FactoryBean主要用于初始化其他Bean，我们可以自己实现一个FactoryBean，然后添加自定义实例化逻辑。在Spring中，AOP、ORM、事务管理等都是依靠FactoryBean的扩展来实现的。

