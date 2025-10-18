---
title: SpringBoot介绍
categories: 
    - Java
date: 2018-10-22 22:40:23
tags:
    - Spring
    - SpringBoot
cover: https://static.jiangliuhong.top/blogimg/spring/springboot.jpg
---

# SpringBoot介绍

> Spring框架为我们提供了多种解决方案，但在使用它的时候总免不了进行导包、配置等操作。于是在2012年10月，有人提出了新需求，要求在Spring框架中支持无容器Web应用程序体系结构，即无不需要将项目打包后放置在中间件中，直接通过main方法引导的Spring容器内配置Web容器服务。 于是，2014年4月,SpringBoot正式发布。

## SpringBoot四大特性

在Spring官网这样说到：

SpringBoot可以轻松创建一个独立的、基于昌平级别的Spring应用程序，我们可以不依赖服务器中间件，直接运行程序。我们的目标是：

- 为所有Spring开发提供一个从根本上更快，且随处可得的入门体验。
- 开箱即用，但通过不采用默认设置可以快速摆脱这种方式。
- 提供一系列大型项目常用的非功能性特征，比如：内嵌服务器，安全，指标，健康检测，外部化配置。
- 绝对没有代码生成，也不需要XML配置。

在我看来SpringBoot其实并不是一个新的框架，它更像是一个总指挥，能够按照我的需求去引入框架，比如，我需要使用SpringMVC，对于SpringBoot而言，我只需要引入一个spring-boot-starter-web，它就会默认去帮我把SpringMVC，tomcat等等都引入到我的工程里。

SpringBoot主要提供了四个特性，也正是这四个特性才能改变开发Spring引用程序的方式：

- SpringBoot Starter：它将常用的依赖分组进行了整合，将其合并到一个依赖中，这样可以一次性添加所有的依赖到项目中，注意，这里一般指的是Maven或Gradle。
- 自动配置：SpringBoot的自动配置特性利用了Spring4对条件华配置的支持，合理地推测应用所需的Bean并自动化配置他们。
- 命令行接口（Command-line interface,即`CLI`）：SpringBoot的CLI发挥了Groovy编程语言的优势，并结合自动配置进一步简化Spring应用的开发。
- Actuator：它为SprigBoot应用添加了一定的管理特性。

## SpringBoot优缺点

### 优点

SpringBoot的优点其实可以看做是基于上文四点衍生出来的。

- 简化依赖，避免了我们手动去配置Maven/Gradle依赖，仅一个starter就能够搞定。
- 利用starter可以达到自动化配置的效果。
- 去除了大量的XML配置文件，采用全注解的方式。
- 舍弃外部的服务器中间件，可以利用其内嵌的tomcat/jetty直接运行。
- 使用CLI可以快速构建SpringBoot程序
- 拥有SpringCloud微服务解决方案

在SpringBoot1.0正式发布之后，2014年年底，Spring团队基于SpringBoot推出SpringCloud，提供一套完整的微服务解决方案。

### 缺点

- 升级难，不能友好的兼容老版本的SpringFramework项目。对于某些项目可能会存在升级的情况，即将老的框架升级为新框架，但如果你想升级使用SpringBoot，我想这应该是一个非常困难的过程。
- 配置服务器服务麻烦。
- 增量更新文件麻烦，因为是jar，如果遇见需要更新包类的一个js、html等文件，则需要先解压后再替换，最后再压缩。

## SpringBoot2.0

SpringBoot2.0是2018年3月发布的版本，该版本基于Spring Framework5.0，与SpringFramework5.0对应的是，SpringBoot2.0同样支持的最低版本为Java8，同时也支持Java9。

**SpringBoot2.0新特性：**

- 修改默认数据库连接池，从tomcat改为HikariCP。
- 优化NOSQL（Redis等）集成方式。
- 升级内嵌容器（Tomcat、Jetty）。
- 适配Spring5.0的WebFlux。
- 增加Quartz自动配置，增加Starter。

当然，对于Spring2.0的新特性远不止于此。本文主要列举了几个较为明显且常见的。

