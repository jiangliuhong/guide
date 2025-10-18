---
title: SpringAOP原理
categories: 
    - Java
    - Spring
date: 2018-10-18 21:52:08
tags:
    - Spring
cover: https://static.jiangliuhong.top/blogimg/spring/spring.png    
---
# SpringAOP原理

## 代理

在熟悉AOP之前我们应该了解一个概念：代理。

代理又分静态代理与动态代理。顾名思义，静态代理的代理关系在编译时就确定了 ，而动态代理的代理关系是在编译期确定的。 

动态代理是Java语言中非常经典的一种设计模式，也是所有设计模式中最难理解的一种。

常见的动态代理为JDK原生动态代理和CGLIB动态代理。 

### 静态代理

>  静态代理实现很简单，但此类代理仅适用于代理类较少的情况。

首先定义个接口Hello.

```java
public interface Hello{
    void say(String name);
}
public class HelloImpl implements Hello{
    @Override
    public void sayHello(String str) {
        System.out.println("Hello: " + str);
    }
}
```

静态代理类：

```java
public class HelloProxy implements Hello{
	private Hello hello = new HelloImpl();
    @Override
    public void sayHello(String str) {
        before();
        hello.say(str);
        after();
    }
    private void before(){
        System.out.println("before");
    }
    private void after(){
        System.out.println("after");
    }
}
```

使用HelloProxy类实现了Hello接口，并且在勾账方法中new 一个HelloImpl类的实例，然后在HelloProxy的say方法中调用hello对象的say方法，然后在其前后分别加上before与after方法，然后可在这两个方法中去实现自己的逻辑，即可实现对HelloImpl的代理了。

### JDK动态代理

对于上述静态代理的实例，如果我们使用动态代理，可以这样做：

在JDK的 java.lang.reflect包下有个Proxy类它正是构造代理类的入口。在Proxy类中有一个创建代理对象的方法：newProxyInstance。

Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler handler) 

该方法的三个参数的意义如下：

- loader：指定代理的类的加载器
- interfaces：代理对象需要实现的接口，可指定多个
- handler：方法调用的实际处理者，代理对象的方法调用都会转发到这里

对于上面的例子，我们只需要定义一个类去实现InvocationHandler接口即可。

```java
public class HelloInvocationHandler implements InvocationHandler {
    /**目标对象*/
    private Object target;
    public HelloInvocationHandler(Object target) {
        this.target = target;
    }
    public <T> T getProxy(){
        return (T)Proxy.newProxyInstance(target.getClass().getClassLoader(),target.getClass().getInterfaces(),this);
    }
    private void before(){
        System.out.println("before");
    }
    private void after(){
        System.out.println("after");
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args);
        after();
        return result;
    }
    public static void main(String[] args) {
        HelloInvocationHandler handler = new HelloInvocationHandler(new HelloImpl());
        Hello proxy = handler.getProxy();
        proxy.say("World");
    }
}
```

其打印结果如下：

```
before
helloWorld
after
```

关于动态代理的总结：

- 代理对象是在程序运行时产生的，而不是编译期
- 对代理对象的所有接口方法调用都会转发到InvocationHandler.invoke()方法
- JDK动态代理是基于接口实现的

JDK动态代理虽然为我们提供了较为友好的代理方式，但是JDK动态代理是基于接口实现的，如果对象没有实现接口，那么就不能使用JDK动态代理了。

### CGLIB动态代理

CGLIB(*Code Generation Library*)是一个基于ASM的字节码生成库，它允许我们在运行时对字节码进行修改和动态生成。CGLIB通过继承方式实现代理，它主要是在运行期间动态生成字节码，从而动态生成代理类。

由于CGLIB是第三方开源项目，所以在使用之前，我们需要引入JAR，对于Maven项目而言只需要在POM文件中加入依赖即可：

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.2.8</version>
</dependency>
```

首先编写一个CGLIB的拦截器

```java
public class CGLIBProxy implements MethodInterceptor {
    public <T> T getProxy(Class<T> clazz){
        return (T) Enhancer.create(clazz,this);
    }
    private void before(){
        System.out.println("before");
    }
    private void after(){
        System.out.println("after");
    }
    @Override
    public Object intercept(Object object, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        before();
        Object result = methodProxy.invokeSuper(object, args);
        after();
        return result;
    }
    public static void main(String[] args) {
        CGLIBProxy proxy = new CGLIBProxy();
        HelloImpl hello = proxy.getProxy(HelloImpl.class);
        hello.say("World BY CGLIB");
    }
}
```

执行结果如下：

```
before
helloWorld BY CGLIB
after
```

intercept方法参数说明:

- object：代理的对象
- method：代理对象的方法信息
- args：待执行方法的参数
- methodProxy：调用方法代理对象

其中，intercept方法为我们传入MethodProxy变量，顾名思义，方法代理，也就是说CGLIB提供的代理方法是方法级别的代理，也就是对方法拦截（方法拦截器）。但是CGLIB无法代理final修饰的方法。

## AOP概念

AOP（Aspect Oriented Programming），即面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。 

AOP一般拥有的功能为：

- 前置增强，Before事件
- 后置增强，After事件
- 环绕增强，Around事件

所谓的Around事件其实就是在方法前后分别加上Before与After事件。

## AspectJ 框架

AspectJ 是一个基于 Java 语言的 AOP 框架，提供了强大的 AOP 功能。

AspectJ 是 Java 语言的一个 AOP 实现，其主要包括两个部分：第一个部分定义了如何表达、定义 AOP 编程中的语法规范，通过这套语言规范，我们可以方便地用 AOP 来解决 Java 语言中存在的交叉关注点问题；另一个部分是工具部分，包括编译器、调试工具等。 

对于AspectJ需要到官网下载对应的插件，如果想要支持AspectJ，就必须在源代码中加入AspectJ关键字(如下面的代码块)，然后通过AspectJ对源代码进行编译。

```java
public aspect TxAspect 
{ 
	// 指定执行 Hello.sayHello() 方法时执行下面代码块
	void around():call(void Hello.sayHello()){
		System.out.println("开始事务 ...");
		//proceed() 代表回调原来指定的方法
		proceed();
		System.out.println("事务结束 ...");
	}
}
```

## Spring AOP 

Spring AOP与 AspectJ 相同的是，Spring AOP 同样需要对目标类进行增强，也就是生成新的 AOP 代理类；与 AspectJ 不同的是，Spring AOP 无需使用任何特殊命令对 Java 源代码进行编译，它采用运行时动态地、在内存中临时生成“代理类”的方式来生成 AOP 代理。 

Spring  AOP中使用了与AspectJ相同的注解，但其功能并未依赖AspectJ的功能。

### Spring AOP实现逻辑

在上文中讲到，Spring在初始化Bean的时候会判断BeanPostProcessor接口，然后根据其实现的方法为Bean实现一些前置、后置操作。同样的，Spring AOP 也是基于BeanPostProcessor实现的。在Spring中，有一个抽象类，名叫AbstractAutoProxyCreator，它实现了BeanPostProcessor接口，在postProcessAfterInitialization方法中，调用wrapIfNecessary方法对Bean进行代理包装。

wrapIfNecessary方法时序图如下：

![springaop.png](https://static.jiangliuhong.top/blogimg/spring/springaop.png)

在上述时序图中可见，Spring会根据代理类的实际情况去动态选择JDK代理与CGLIB代理，其中createAopProxy源码如下：

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```

通过源码可以看出，Spring选择JDK代理的条件为：

- 代理的Bean实现了接口
- 没有为Bean指定直接代理

反之，Spring则会选择CGLIB代理。

在Spring为Bean创建好代理对象后，我们在调用Bean时，首先Spring会找到代理对象的invoke方法，然后在该方法中会去查找拦截器，然后执行拦截器方法，最后才执行Bean的方法。

AOP执行代理Bean时序图（以JDK代理方式为例）：

![AOP执行代理Bean时序图](https://static.jiangliuhong.top/blogimg/spring/aopInterceptors.png)

## 总结

在本文中阐述了三种实现代理的模式，即静态代理、JDK动态代理、CGLIB动态代理。

静态代理实现简单，缺点是不能扩展，仅适用于类较少，变化较少的功能。

JDK动态代理扩展性强，缺点是必须实现接口。

CGLIB动态代理扩展性强，基于方法拦截实现动态代理，不用实现接口，直接对类进行代理。

AOP是具有横切性质的系统服务，AOP的出现是对OOP的良好补充，它使得开发者能用更优雅的方式处理具有横切性质的服务。

AspectJ 是在系统编译时决定代理关系。

SpringAOP是在Spring容器加载过程中对Bean进行处理生成代理类，因为SpringAOP每次运行时都会产生一个AOP代理类，因此性能较AspectJ 略差一筹。



