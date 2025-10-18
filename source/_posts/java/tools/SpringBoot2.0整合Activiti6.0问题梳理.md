---
title: SpringBoot2.0整合Activiti6.0问题梳理
date: 2019-05-23 09:49:02
categories: Java
tags: 
 - tools
---
# SpringBoot2.0整合Activiti6.0问题梳理

SpringBoot整合Activiti很简单，我们可以通过springboot的starter来快速整合，只需要在pom文件中引入一下内容即可：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>6.0.0</version>
</dependency>
```

详细源码请见:[https://github.com/jiangliuhong/springcloud-stu/tree/master/scs-act](https://github.com/jiangliuhong/springcloud-stu/tree/master/scs-act)



在Activiti6.0发布的时候，SpringBoot2.0还未发布，所以直接启动，会出现如下错误:

```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'requestMappingHandlerMapping' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Invocation of init method failed; nested exception is java.lang.ArrayStoreException: sun.reflect.annotation.TypeNotPresentExceptionProxy
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1706) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:579) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:501) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:317) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:228) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:315) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:199) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:760) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:869) ~[spring-context-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:550) ~[spring-context-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:140) ~[spring-boot-2.0.2.RELEASE.jar:2.0.2.RELEASE]
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:759) [spring-boot-2.0.2.RELEASE.jar:2.0.2.RELEASE]
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:395) [spring-boot-2.0.2.RELEASE.jar:2.0.2.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:327) [spring-boot-2.0.2.RELEASE.jar:2.0.2.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1255) [spring-boot-2.0.2.RELEASE.jar:2.0.2.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1243) [spring-boot-2.0.2.RELEASE.jar:2.0.2.RELEASE]
	at pers.jarome.scs.act.ActApplication.main(ActApplication.java:17) [classes/:na]
Caused by: java.lang.ArrayStoreException: sun.reflect.annotation.TypeNotPresentExceptionProxy
	at sun.reflect.annotation.AnnotationParser.parseClassArray(AnnotationParser.java:724) ~[na:1.8.0_181]
	at sun.reflect.annotation.AnnotationParser.parseArray(AnnotationParser.java:531) ~[na:1.8.0_181]
	at sun.reflect.annotation.AnnotationParser.parseMemberValue(AnnotationParser.java:355) ~[na:1.8.0_181]
	at sun.reflect.annotation.AnnotationParser.parseAnnotation2(AnnotationParser.java:286) ~[na:1.8.0_181]
	at sun.reflect.annotation.AnnotationParser.parseAnnotations2(AnnotationParser.java:120) ~[na:1.8.0_181]
	at sun.reflect.annotation.AnnotationParser.parseAnnotations(AnnotationParser.java:72) ~[na:1.8.0_181]
	at java.lang.Class.createAnnotationData(Class.java:3521) ~[na:1.8.0_181]
	at java.lang.Class.annotationData(Class.java:3510) ~[na:1.8.0_181]
	at java.lang.Class.createAnnotationData(Class.java:3526) ~[na:1.8.0_181]
	at java.lang.Class.annotationData(Class.java:3510) ~[na:1.8.0_181]
	at java.lang.Class.getAnnotation(Class.java:3415) ~[na:1.8.0_181]
	at java.lang.reflect.AnnotatedElement.isAnnotationPresent(AnnotatedElement.java:258) ~[na:1.8.0_181]
	at java.lang.Class.isAnnotationPresent(Class.java:3425) ~[na:1.8.0_181]
	at org.springframework.core.annotation.AnnotatedElementUtils.hasAnnotation(AnnotatedElementUtils.java:573) ~[spring-core-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.isHandler(RequestMappingHandlerMapping.java:177) ~[spring-webmvc-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.initHandlerMethods(AbstractHandlerMethodMapping.java:217) ~[spring-webmvc-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.afterPropertiesSet(AbstractHandlerMethodMapping.java:188) ~[spring-webmvc-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.afterPropertiesSet(RequestMappingHandlerMapping.java:129) ~[spring-webmvc-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1765) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1702) ~[spring-beans-5.0.6.RELEASE.jar:5.0.6.RELEASE]
	... 16 common frames omitted
```

查阅网上资料得知是因为，Activiti6.0发布时，SpringBoot2.0 并没有发布，所以Activiti6.0仅支持1.2.6以上，2.0.0以下版本的SpringBoot。

通过调试（调试方式可以参照这一篇博客：[深入Spring Boot：怎样排查 java.lang.ArrayStoreException](http://hengyunabc.github.io/spring-boot-ArrayStoreException/)）查看源码很容易发现`SecurityAutoConfiguration`源码因为SpringBoot内部结构的变化，从而引起该类出现编译错误。

针对这一类情况有三种解决办法，分别如下：

- 将springboot2.0换成1.X版本
- 在springboot启动类上排除`SecurityAutoConfiguration`类
- 修改SecurityAutoConfiguration源码，使其支持SpringBoot2.0

对于上诉的三种方法，第一种最为简单，直接切换版本即可，但这样就不能使用SpringBoot2.0的特性，所以并不推荐；对于第三种方法，我认为难度较大，且具备极大的风险，并且Activiti的更高版本肯定也会修复，所以也不推荐该方法。

所以，最优的方法当属第二种，其具体操作如下：

```
@SpringBootApplication(exclude = org.activiti.spring.boot.SecurityAutoConfiguration.class)
public class ActApplication {
    public static void main(String[] args) {
        SpringApplication.run(ActApplication.class);
    }
}
public class ActApplication {
    public static void main(String[] args) {
        SpringApplication.run(ActApplication.class);
    }
}
```

但是，如果你的启动类中加入了`@EnableAutoConfiguration`注解，上面的方法就失效了，此时应该使用下面的方式：

```
@SpringBootApplication
@EnableAutoConfiguration(exclude =   org.activiti.spring.boot.SecurityAutoConfiguration.class)
public class ActApplication {
    public static void main(String[] args) {
        SpringApplication.run(ActApplication.class);
    }
}
```

详细源码请见:[https://github.com/jiangliuhong/springcloud-stu/tree/master/scs-act](https://github.com/jiangliuhong/springcloud-stu/tree/master/scs-act)
