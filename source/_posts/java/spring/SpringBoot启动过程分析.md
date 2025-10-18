---
title: SpringBoot启动过程分析
categories: 
    - Java
date: 2021-03-18 11:40:23
tags:
    - Spring
    - SpringBoot
cover: https://static.jiangliuhong.top/blogimg/spring/springboot.jpg
top_img: https://static.jiangliuhong.top/images/2021/3/kjdq.jpg
---

# SpringBoot启动过程分析

> SpringBoot的出现给我们带了许多的便利性，其中一点就是可以内置tomcat，从而实现从jar包直接运行，那么SpringBoot是怎么实现的呢？

## 嵌入式tomcat

在一个简单的`SpringBoot`项目中，我们只需要在项目中添加`spring-boot-starter-web`依赖，然后通过`SpringApplication.run`方法就可以启动一个`web`服务。查看`spring-boot-starter-web`的依赖关系，发现其依赖了`tomcat-embed-core`，其依赖图如下：

![](https://static.jiangliuhong.top/images/2021/3/1616049403075.png)

关于`tomcat-embed-core`，的理解，就像他的名字`embed`一样，是一个嵌入式的tomcat，他无需构建`WAR`包到`Tomcat`服务器中，在项目中引入了这个依赖之后，可以通过该jar包提供的api从而实现一个简单的web服务，示例代码如下：

在pom中引入：

```xml
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-core</artifactId>
    <version>8.5.35</version>
</dependency>
```

编写启动类：

```java
package top.jiangliuhong;

import org.apache.catalina.Context;
import org.apache.catalina.LifecycleException;
import org.apache.catalina.startup.Tomcat;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.File;
import java.io.IOException;
import java.io.Writer;

public class EmbedTomcatTest {
    public static void main(String[] args) throws LifecycleException {
        Tomcat tomcat = new Tomcat();
        tomcat.setPort(8081);
        final Context context = tomcat.addContext("/", new File(".").getAbsolutePath());
        Tomcat.addServlet(context, "MVC", new HttpServlet() {
            @Override
            protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
                Writer w = resp.getWriter();
                w.write("Embedded Tomcat servlet.\n");
                w.flush();
                w.close();
            }
        });
        context.addServletMappingDecoded("/*", "MVC");
        tomcat.start();
        tomcat.getServer().await();
    }
}
```

访问`8081`端口效果如下：

![](https://static.jiangliuhong.top/images/2021/3/1616056802193.png)

这里我使用的是`addContext`方法，其作用类似tomcat配置文件`server.xml`中的`Context`属性；同时这里我是使用直接new了一个HttpServlet作为返回，当然也可以使用`addWebapp`方法，将一个目录加载到tomcat中，类似于tomcat安装目录下的`webapps`目录。

## SpringMVC去XML

SpringBoot的宗旨是去XML化，在以往项目中，都是配置一个spring的xml文件，然后再在`web.xml`中配置对应的内容，那么SpringBoot是怎么实现去掉XML呢？结合spring官网对`SpringMVC`的描述可以很好解释这个问题，官网地址为：[https://docs.spring.io/spring-framework/docs/current/reference/html/web.html](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html)

在spring官网中这样说道，在Servlet 3.0+环境中，您可以选择以编程方式配置Servlet容器，以替代方式或与`web.xml`文件结合使用，示例代码如下：

```java
import org.springframework.web.WebApplicationInitializer;

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

`WebApplicationInitializer `是`SpringMVC`提供的接口，在`servlet`初始化的时候会调用该接口的`onStartup`方法。其加载的类为`org.springframework.web.SpringServletContainerInitializer`,`ServletContainerInitializer`这是servlet3.0新特性提供的接口，servlet3.0是这样规定的：

- 在servlet启动时，会读取`META-INF/services/javax.servlet.ServletContainerInitializer`文件，该文件内容为`ServletContainerInitializer`的实现类的全路径
- 在`ServletContainerInitializer`的实现类中可以添加`@HandlesTypes`注解，然后servlet启动是，会自动扫描对应的类，然后作为`onStartup`的参数

`SpringServletContainerInitializer`配置如下：

![](https://static.jiangliuhong.top/images/2021/3/1616058617497.png)

在onStartup中，循环实例化`webAppInitializerClasses`并调用其onStartup方法

```java
servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
AnnotationAwareOrderComparator.sort(initializers);
for (WebApplicationInitializer initializer : initializers) {
    initializer.onStartup(servletContext);
}
```

由此可见，基于上述特性，springboot就能够实现`SrpingMVC`的去XML化,当然不局限于`SpringMVC`，任意servlet的框架都可以基于3.0的新特性实现去XML化

## SpringBoot的run方法

在前面熟悉了嵌入式的tomcat与servlet的去XMl特性后就可以进入本次的主题了，SpringBoot是如何如何通过内置的tomcat启动项目的，在启动SpringBoot项目的时候，一般会是如下代码：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class UaaApplication {
    public static void main(String[] args) {
        SpringApplication.run(UaaApplication.class);
    }
}
```

`SpringApplication.run()`方法主要流程如下：

```java
public ConfigurableApplicationContext run(String... args) {
    // 开启一个定时器
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    // 获取监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 执行Spring应用刚启动时的前置事件
    listeners.starting();
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 创建并配置环境变量
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 配置忽略BeanInfo
        configureIgnoreBeanInfo(environment);
        // 获取打印Banner对象
        Banner printedBanner = printBanner(environment);
        // 创建上下文对象，根据项目类型创建不同的，主要为Servlet与Reactive(WebFlux)
        context = createApplicationContext();
        // 加载spring.factories文件
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                         new Class[] { ConfigurableApplicationContext.class }, context);
        // 上下文准备工作，主要为上下文的后置处理、注册banner对象、懒加载处理等
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 刷新上下文，springboot启动的核心方法
        refreshContext(context);
        // 刷新上下文的后置方法
        afterRefresh(context, applicationArguments);
        // 停止计时器
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 执行Spring应用正在运行事件
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

### refreshContext

该方法是整个项目启动的核心方法，其执行对象源于`createApplicationContext`,如果当前应用类型为`Servlet`，其返回的上下文类型为：`org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext`，不过最终起作用的还是抽象类里的`refresh`方法`org.springframework.context.support.AbstractApplicationContext#refresh`。

在`AbstractApplicationContext`类中提供了`onRefresh`方法，其作用是在特定的上下文子类中初始化其他特殊操作。在`ServletWebServerApplicationContext`类中重写了该方法，其内容如下：

```java
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}
```

其中`createWebServer`核心的方法又为`factory.getWebServer(getSelfInitializer())`，该方法会根据当前上下文环境去获取不同的servlet服务器工厂(`org.springframework.boot.web.servlet.server.ServletWebServerFactory`)。

servlet服务器工厂列表如下：

- JettyServletWebServerFactory
- TomcatServletWebServerFactory
- UndertowServletWebServerFactory

在`ServletWebServerFactory`方法中，定义了`getWebServer`，即嵌入式服务器加载方法，在`TomcatServletWebServerFactory`工厂的实现逻辑如下：

```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
    if (this.disableMBeanRegistry) {
        Registry.disableRegistry();
    }
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    Connector connector = new Connector(this.protocol);
    connector.setThrowOnFailure(true);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    configureEngine(tomcat.getEngine());
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    prepareContext(tomcat.getHost(), initializers);
    return getTomcatWebServer(tomcat);
}
```

其中`prepareContext`主要讲tomcat的上下文和spring的上下文合并，其主要步骤为：

- 创建一个`TomcatEmbeddedContext`
- 设置一些`TomcatEmbeddedContext`必要的属性
- 通过`mergeInitializers`方法合并方法传入`ServletContextInitializer`
    - 方法传入的`ServletContextInitializer[] initializers`为Spring的上下文
- 在`configureContext`方法中将上下文与tomcat进行绑定

到此，内置tomcat启动就已经完成了。

### 构建并部署war


通过官方文档，如果需要构建并部署war文件时，需要使用`WebApplicationInitializer`，由于该类是个抽象类，所以需要继承该类;当然也可以直接实现`WebApplicationInitializer`接口


## 补充说明

### `getSpringFactoriesInstances`说明

该方法的源码为：

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

其中比较重要的方法是`SpringFactoriesLoader.loadFactoryNames(type, classLoader)`,通过该方法能够定位到`loadSpringFactories`方法，熟悉SpringBoot的starter工作原理的应该知道，在每个starter包里都有`META-INF/spring.factories`文件，而`loadSpringFactories`的作用就是加载这些文件为自动装配服务。

### `SpringApplicationRunListener`方法说明

```java
public interface SpringApplicationRunListener {

	/**
	 * 运行时机： Spring应用刚启动
	 */
	void starting();

	/**
	 * 运行时机：ConfigurableEnvironment准备妥当，允许将其调整
	 */
	void environmentPrepared(ConfigurableEnvironment environment);

	/**
	 * 运行时机：ConfigurableApplicationContext准备妥当，允许将其调整
	 */
	void contextPrepared(ConfigurableApplicationContext context);

	/**
	 * 运行时机：ConfigurableApplicationContext已经装载，但仍未启动
	 */
	void contextLoaded(ConfigurableApplicationContext context);

	/**
	 * 运行时机：ConfigurableApplicationContext已启动，此时Spring Bean已经初始化完成
	 */
	void started(ConfigurableApplicationContext context);

	/**
	 * 运行时机：Spring应用正在运行
	 */
	void running(ConfigurableApplicationContext context);

	/**
	 * 运行时机： Spring应用运行失败
	 */
	void failed(ConfigurableApplicationContext context, Throwable exception);

}
```

