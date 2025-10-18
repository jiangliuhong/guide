---
title: SpringBoot启动过程源码分析
categories: 
    - Java
date: 2018-10-22 22:40:23
tags:
	- Spring
	- SpringBoot
cover: https://static.jiangliuhong.top/blogimg/spring/springboot.jpg
---
# SpringBoot启动过程源码分析

> 随着SpringBoot的热度越来越高，现在企业中对SpringBoot的使用也越来越频繁，而SpringBoot也没让我们失望，它极大的提高了编程的快捷性，今天就SpringBoot(1.5.8.RELEASE)启动源码来看看SpringBoot是如何避繁就简的吧。

## 启动入口

SpringBoot为我们提供了一个简单快捷的启动方式，当我们需要更多功能时，只需要通过在`DemoApplication `类上增加相应的注解即可：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

此处简单描述一下`@SpringBootApplication`注解：

此注解注解在Spring Boot的XXXApplication类（有main函数，程序启动的入口）上，其结合了三个注解: @Configuration, @EnableAutoConfiguration 和 @ComponentScan。 

**@Configuration**：

 `@Configuration` 中所有带 `@Bean` 注解的方法都会被动态代理，因此调用该方法返回的都是同一个实例。 

**@EnableAutoConfiguration**：

使用该注解后，Spring Boot将尝试根据项目引入的依赖来配置程序 。

**@ComponentScan**：

指定要扫描的包，以及扫描的条件，默认扫描@ComponentScan注解所在类的同级类和同级目录下的所有类。

另外官方具有很详细的说明，请查阅：[https://docs.spring.io/spring-framework/docs/4.3.5.RELEASE/javadoc-api/org/springframework/context/annotation/](https://docs.spring.io/spring-framework/docs/4.3.5.RELEASE/javadoc-api/org/springframework/context/annotation/)

## SpringApplication.run方法

通过跟读代码发现，SpringApplication.run方法主要分为两个步骤，`new SpringApplication(sources)`,`run`：

```java
/**
	 * Static helper that can be used to run a {@link SpringApplication} from the
	 * specified sources using default settings and user supplied arguments.
	 * @param sources the sources to load
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return the running {@link ApplicationContext}
	 */
	public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
		return new SpringApplication(sources).run(args);
	}
```

### new SpringApplication(sources)

首先看看`new SpringApplication(sources)`方法

通过调试发现，该方法的核心就是`initialize`，其源码如下：

```java
private void initialize(Object[] sources) {
		if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
    	//推论当前是否为Web运行环境
		this.webEnvironment = deduceWebEnvironment();
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
    	//初始化监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

其中比较疑惑的是`deduceWebEnvironment`方法，因为`WEB_ENVIRONMENT_CLASSES`的值是不变的，所以该方法返回值恒为`true`。

`setListeners`方法主要是将`getSpringFactoriesInstances`返回的实例加入到`listeners`集合中。

`getSpringFactoriesInstances`源码如下：

```java
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
    	//加载出需要实例化的类
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    	//创建相应的实例
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
```

`SpringFactoriesLoader.loadFactoryNames`源码如下：

```java
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		try {
            //获取jar包中的资源，其中FACTORIES_RESOURCE_LOCATION的值为META-INF/spring.factories
			Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			List<String> result = new ArrayList<String>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
				String factoryClassNames = properties.getProperty(factoryClassName);
				result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
			}
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
					"] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

通过上诉代码中的`classLoader.getResources(FACTORIES_RESOURCE_LOCATION)`代码，我们可以看出SpringBoot去加载了jar包中META-INF/spring.factories文件，那么这个文件又具有怎样的作用呢？

官方给出的说明是：

使用给定的类加载器从META-INF / spring.factories加载给定类型的工厂实现的完全限定类名。 

翻阅了`spring-test-4.3.12.RELEASE`下的`spring.factories`文件，其源码如下：

```xml
# Default TestExecutionListeners for the Spring TestContext Framework
#
org.springframework.test.context.TestExecutionListener = \
	org.springframework.test.context.web.ServletTestExecutionListener,\
	org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener,\
	org.springframework.test.context.support.DependencyInjectionTestExecutionListener,\
	org.springframework.test.context.support.DirtiesContextTestExecutionListener,\
	org.springframework.test.context.transaction.TransactionalTestExecutionListener,\
	org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener

# Default ContextCustomizerFactory implementations for the Spring TestContext Framework
#
org.springframework.test.context.ContextCustomizerFactory = \
	org.springframework.test.context.web.socket.MockServerContainerContextCustomizerFactory
```

不难发现，`spring.factories`中配置的这些类，主要作用是告诉Spring Boot这个stareter所需要加载的那些xxxAutoConfiguration类，也就是你真正的要自动注册的那些bean或功能。然后，我们实现一个spring.factories指定的类，标上@Configuration注解，一个starter就定义完了。

如果想看一下SpringBoot启动过程中都加载了那些spring.factories文件，可以写这样一个测试类：

```java
@SpringBootTest
public class SpringTest {

    public static String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

    @Test
    public void test1(){
        try {
            //EurekaServerApp 应用程序入口
            EurekaServerApp app = new EurekaServerApp();
            Enumeration<URL> urls =app.getClass().getClassLoader().getResources(FACTORIES_RESOURCE_LOCATION);
            while(urls.hasMoreElements()){
                URL url = urls.nextElement();
                System.out.println(url.toString());
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

其打印结果如下：

```
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/cloud/spring-cloud-starter-eureka-server/1.4.0.RELEASE/spring-cloud-starter-eureka-server-1.4.0.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/cloud/spring-cloud-context/1.3.0.RELEASE/spring-cloud-context-1.3.0.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/cloud/spring-cloud-commons/1.3.0.RELEASE/spring-cloud-commons-1.3.0.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/cloud/spring-cloud-netflix-eureka-server/1.4.0.RELEASE/spring-cloud-netflix-eureka-server-1.4.0.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/boot/spring-boot-actuator/1.5.8.RELEASE/spring-boot-actuator-1.5.8.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/cloud/spring-cloud-netflix-core/1.4.0.RELEASE/spring-cloud-netflix-core-1.4.0.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/cloud/spring-cloud-netflix-eureka-client/1.4.0.RELEASE/spring-cloud-netflix-eureka-client-1.4.0.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/boot/spring-boot-test/1.5.8.RELEASE/spring-boot-test-1.5.8.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/boot/spring-boot/1.5.8.RELEASE/spring-boot-1.5.8.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/spring-beans/4.3.12.RELEASE/spring-beans-4.3.12.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/boot/spring-boot-test-autoconfigure/1.5.8.RELEASE/spring-boot-test-autoconfigure-1.5.8.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/boot/spring-boot-autoconfigure/1.5.8.RELEASE/spring-boot-autoconfigure-1.5.8.RELEASE.jar!/META-INF/spring.factories
jar:file:/D:/develop/soft/maven/repo/repositories/org/springframework/spring-test/4.3.12.RELEASE/spring-test-4.3.12.RELEASE.jar!/META-INF/spring.factories
```

### run

在run方法中，首先定义了一个`StopWatch`来标记启动时间。

源码如下：

```java
public ConfigurableApplicationContext run(String... args) {
    	//初始化一个计时器
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
    	//设置java.awt.headless系统属性为true。
		configureHeadlessProperty();
    	//获取SpringApplicationRunListeners
		SpringApplicationRunListeners listeners = getRunListeners(args);
		//开始执行监听器
    	listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
             //在上下文创建之前准备环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
             //准备Banner打印器
			Banner printedBanner = printBanner(environment);
             //创建上下文
			context = createApplicationContext();
			analyzers = new FailureAnalyzers(context);
			//上下文前置处理
             prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
             //刷新上下文
			refreshContext(context);
             //上下文后置处理
			afterRefresh(context, applicationArguments);
             //发出结束执行的事件
			listeners.finished(context, null);
             //停止计时器
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```

#### configureHeadlessProperty  

查询该方法源码，发现其是将`java.awt.headless`的值设置为`true`，查阅资料发现`Headless`模式是在缺少显示屏、键盘或者鼠标是的系统配置，因为服务器（如提供Web服务的主机）往往可能缺少前述设备，但又需要使用他们提供的功能，生成相应的数据，以提供给客户端（如浏览器所在的配有相关的显示设备)、键盘和鼠标)的主机），所以需要依靠系统的计算能力模拟出这些特性来，即设置其值为`true`。

#### getRunListeners

获取SpringApplicationRunListeners，这里的实现方式与上文中的`SpringFactoriesLoader.loadFactoryNames`类似。

#### listeners.starting

> 该方法其实是遍历在最初加入进去的监听器，然后分别执行各监听器对应的`staring`方法

通过下面这个类图，我们不难发现`listeners.starting`最后执行的是`multicastEvent`方法。

![EventPublishingRunListener](https://static.jiangliuhong.top/blogimg/spring/EventPublishingRunListener.png)

 `multicastEvent`方法源码如下：

```java
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(new Runnable() {
					@Override
					public void run() {
						invokeListener(listener, event);
					}
				});
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

通过上述源码继续我们依次往下跟读，其方法顺序如下`invokeListener`=>`doInvokeListener`=>`onApplicationEvent`，`onApplicationEvent`方法则是每个监听器真实是生效的方法。

#### prepareEnvironment

该方法会根据上文中获取的`SpringApplicationRunListeners`以及springboot启动时传入的参数来构建启动环境。

prepareEnvironment源码如下：

```java
private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		configureEnvironment(environment, applicationArguments.getSourceArgs());
    	//在上下文创建之前准备环境
		listeners.environmentPrepared(environment);
		if (!this.webEnvironment) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertToStandardEnvironmentIfNecessary(environment);
		}
		return environment;
	}
```

**configureEnvironment**

查看该方法不难发现该方法主要是通过`configurePropertySources`配置Property Sources，通过`configureProfiles`配置Profiles。

#### createApplicationContext

> 创建Spring上下文

这个方法有点的源码很简单，就实例化了一个`org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext`。但是如果想要看懂创建上下的实现逻辑，需要去研究spring的源码，这里就不做描述了。

我在idea中查看了一下这个`AnnotationConfigEmbeddedWebApplicationContext`类的继承关系，现将该图放置在下方。后面有时间再去研究吧。

![AnnotationConfigEmbeddedWebApplicationContext](https://static.jiangliuhong.top/blogimg/spring/AnnotationConfigEmbeddedWebApplicationContext.png)

#### prepareContext上下文前置处理

源码如下：

```java
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
    	//将上下文与环境关联上
		context.setEnvironment(environment);
		//申请任何相关的后期处理ApplicationContext
    	//将一些生成器加入到spring上下文中
    	postProcessApplicationContext(context);
    	//初始化，ApplicationContextInitializer在刷新之前将任何应用于上下文。
		applyInitializers(context);
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
             //记录启动信息
			logStartupInfo(context.getParent() == null);
             //记录活动的配置文件信息
			logStartupProfileInfo(context);
		}

		// Add boot specific singleton beans
		context.getBeanFactory().registerSingleton("springApplicationArguments",
				applicationArguments);
		if (printedBanner != null) {
			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
		}

		// Load the sources
		Set<Object> sources = getSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[sources.size()]));
		listeners.contextLoaded(context);
	}
```

## 总结

SpringBoot启动主要分为两个步骤：

1. SpringApplication实例的构建过程：其中最为主要的是通过META-INF/spring.factories加载监听器。
2. SpringApplication实例run方法的执行过程：其中主要有一个SpringApplicationRunListeners的概念，它作为Spring Boot容器初始化时各阶段事件的中转器，将事件派发给感兴趣的Listeners(在SpringApplication实例的构建过程中得到的)。这些阶段性事件将容器的初始化过程给构造起来，提供了比较强大的可扩展性。 

 

 