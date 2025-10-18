---
title: SpringApplicationContext初始化过程
categories: 
    - Java
    - Spring
date: 2018-10-18 20:52:08
tags:
    - Spring
cover: https://static.jiangliuhong.top/blogimg/spring/spring.png    
---

# SpringApplicationContext初始化过程

## ContextLoaderListener

在SpringBoot面世之前。在一般的WEB项目中，项目的启动都是从web.xml开始的，如果我们想在项目中使用Spring，只需在web.xml文件中指定以下内容即可：

```
<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>classpath:applicationContext.xml</param-value>
</context-param>
<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

通过以上代码片段不难看出Spring正是通过ContextLoaderListener监听器来进行容器初始化的，查看`ContextLoaderListener`源码：

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener() {
	}
    public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}
	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}
}
```

根据该类中的注释可以看出initWebApplicationContext方法为核心的初始化方法，从initWebApplicationContext方法源代码可以看出Spring初始化容器主要分为以下几个步骤：

1. 创建容器WebApplicationContext
2. 验证当前容器是否为可配置的，是则配置并且刷新当前容器 
3. 将当前创建的容器设置到servlet上下文中

## SpringBoot中的Spring

> 上文根据一般WEB项目跟踪了Spring容器初始化过程，但是从上诉过程并不能相对明显地看出Spring容器初始化过程。

在SpringBoot面世后，它简化了许多的配置方式，在SpringBoot中只需引入相应的start即可使用Spring，接下来就去看看SpringBoot中的Spring吧。

通过SpringBoot入口方法`SpringApplication.run`可以看到以下代码：

```java
ApplicationArguments applicationArguments = new DefaultApplicationArguments(
    args);
ConfigurableEnvironment environment = prepareEnvironment(listeners,
                                                         applicationArguments);
configureIgnoreBeanInfo(environment);
Banner printedBanner = printBanner(environment);
//创建容器
context = createApplicationContext();
exceptionReporters = getSpringFactoriesInstances(
    SpringBootExceptionReporter.class,
    new Class[] { ConfigurableApplicationContext.class }, context);
//容器准备工作
prepareContext(context, environment, listeners, applicationArguments,
               printedBanner);
//刷新容器
refreshContext(context);
```

其中，创建容器容器的方法是createApplicationContext，createApplicationContext方法会根据当前启动类型去初始化不同的Spring容器，主要类型为以下三种：

- NONE：非WEB，普通应用程序
- REACTIVE：反应堆栈Web容器(5.x新加)
- SERVLET：Web容器

ps:反应堆栈Web容器，即WebFlux框架，该框架是Spring 5.x新加的框架，详细内容请访问SpringCloud中文网：[https://springcloud.cc/web-reactive.html](https://springcloud.cc/web-reactive.html)

### prepareContext

prepareContext方法是做context的准备工作，该方法主要对容器进行一些预设置，源码中，该方法中的postProcessApplicationContext方法向beanFactory中添加了一个beanNameGenerator：

```java
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
		if (this.beanNameGenerator != null) {
			context.getBeanFactory().registerSingleton(
					AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
					this.beanNameGenerator);
		}
		if (this.resourceLoader != null) {
			if (context instanceof GenericApplicationContext) {
				((GenericApplicationContext) context)
						.setResourceLoader(this.resourceLoader);
			}
			if (context instanceof DefaultResourceLoader) {
				((DefaultResourceLoader) context)
						.setClassLoader(this.resourceLoader.getClassLoader());
			}
		}
	}
```

其中，BeanNameGenerator用来生成扫描到的Bean在容器中的名字。

在prepareContext方法中，applyInitializers也是一个颇为重要的内容，通过查询资料发现该方法主要是对已创建的并且未被刷新的容器进行设置的自定义应用上下文初始化器。

### refreshContext

通过跟踪refreshContext方法不难发现，其最终执行的是AbstractRefreshableApplicationContext类中的refresh方法，其源码如下：

```java
public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        synchronized(this.startupShutdownMonitor) {
         // 为应用上下文的刷新做准备--设置时间、记录刷新日志、初始化属性源中的占位符(事实上什么都没做)和验证必 要的属性等
            this.prepareRefresh();
            // 让子类刷新内部的bean factory
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            //为上下文准备bean factory
            this.prepareBeanFactory(beanFactory);

            try {
                // bean factory 后置处理
                this.postProcessBeanFactory(beanFactory);
                // 调用应用上下文中作为bean注册的工厂处理器
                this.invokeBeanFactoryPostProcessors(beanFactory);
                // 注册拦截创建bean的bean处理器
                this.registerBeanPostProcessors(beanFactory);
                // 初始化消息源
                this.initMessageSource();
                // 初始化事件广播
                this.initApplicationEventMulticaster();
                // 初始化特定上下文子类中的其它bean
                this.onRefresh();
                // 注册监听器bean
                this.registerListeners();
                 // 实例化所有的单例bean
                this.finishBeanFactoryInitialization(beanFactory);
                 // 发布相应的事件
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }
			 //销毁错误的资源
                this.destroyBeans();
                //重置刷新标志
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }
```

从以上代码的注释，可以看出refresh方法是Spring容器初始化的过程中加载Bean至关重要的一环，其职责主要是获取Bean，并初始化Bean。