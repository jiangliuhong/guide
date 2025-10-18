---
title: SpringFramework历史版本
categories: 
    - Java
    - Spring
date: 2018-10-18 21:52:08
tags:
    - Spring
cover: https://static.jiangliuhong.top/blogimg/spring/spring.png   
---
# SpringFramework历史版本

> 对于Spring而言，迄今已有14年历史了，版本也到达了5.0，作为JavaWEB开发领域的常青树，现在Spirng已不再简单是一个框架了，在Spring的项目中主要有：SpringFramework(也就是我们常说的Spring，主要有IOC、AOP等)、SpringBoot、SpringCloud、SpringData、SpringIO等等。详情请见官网：[spring.io/projects](spring.io/projects)
>
> 本文主要描述SpringFrameworkd、SpringBoot、SpringCloud版本历史
>
> SpringFramework，一下简称Spring
>
> Spring框架是由大量的模块组成，其中主要包括：Core、Beans、Context、AOP、Web、ORM、JDBC等等。在这些组件中，主要以Core、Beans、Context为核心，Spring框架通过该组件实现依赖注入与控制反转，使得设计和测试松散耦合，极大提高了编程效率。

## Spring版本情况

### Spring雏形

> 2002年10月,Rod Johnson发布《Expert One-on-One J2EE设计和开发》一书

在Spring框架面世之前，当时在JavaEE开发中基本都是使用EJB框架进行，但可能是EJB设计太过庞大、繁重，又或是EJB发展的进度追不上时代的潮流，在2002年10月，Rod Johnson撰写了一本名为《Expert One-on-One J2EE设计和开发》的书。本书主要概括了当时Java企业应用程序开发的现状已经指出了JavaEE和EJB框架的缺陷，并且本书基于普通Java类和依赖注入提出了更为简单的解决方案。

### Spring1.0

> 2004年3月，Spring1.0发布

2003年6月，Spring Framework 第一次以 Apache 2.0 许可证下发布0.9版本，2004年3月，Spring1.0正式发布

对于Spring1.0，其源码只有一个包，在该包中包含了aop、beans、context、core、jdbc、orm等。对于此时的版本，Spring1.0仅支持XML配置的方式。

### Spring2.0

> 2006年10 月，Spring2.0发布

对于2.0，Spring主要增加了对注解的支持，实现了基于注解的配置。

在2007年11月，发布Spring2.5，该版本具备的特性有：

- 添加可扩展的XML配置功能，用于简化XML配置
- 支持Java5
- 添加额外的IOC容器扩展点，支持动态语言（如groovy，aop增强功能和新的bean范围 ）

### Spring3.0

> 2009年12月，Spring3.0发布

Spring3.0主要具有的特性有：

- 模块重组系统
- 支持Spring表达式语言（Spring Expression）
- 基于Java的Bean配置（JavaConfig）
- 支持嵌入式数据库：HSQL、H2等
- 支持REST
- 支持Java6

### Spring4.0

> 2013年12月，发布Spring4.0

对于Spring4.0是Spring版本历史上的一重大升级。其特性为：

- 全面支持Java8
  - 支持Lambda表达式
  - 支持Java8的时间和日期API
  - 支持重复注解
  - 支持Java8的Optional
- 核心容器增强
  - 增加泛型依赖注入
  - 增加Map依赖注入
  - 增加List依赖注入
  - 支持lazy注解配置懒加载
  - 支持Condition条件注解
  - CGLIB动态代理增强
- 支持基于GroovyDSL定义Bean
- Web增强
  - 增强SpringMVC，基于Servlet3.0开发
  - 提供RestController注解
  - 提供AsyncRestTemplate支持客户端的异步无阻塞请求
- 增加对WebSocket的支持

### Spring5.0

> 2017年9月，Spring5.0发布

Spring5.0特性如下：

- 升级到Java8、JavaEE7
  - 废弃低版本，将Java8、JavaEE 7作为最低版本要求
  - 兼容Java9
  - 兼容JavaEE8
- 反应式编程模型，增加WebFlux模块
- 升级SpringMVC，增加对最新的API（Jackson等）的支持
- 增加函数式编程模式
- 重构源码，部分功能使用Lambda表达式实现

## Spring框架子项目

对于Spring而言，Spring框架在其生态环境下极其重要的一环。而在Spring框架中所含的子项目也多不胜数，下面贴一张互联网上的图片对每个子项目做个简单介绍。

![springchild.jpeg](https://static.jiangliuhong.top/blogimg/spring/springchild.jpeg)