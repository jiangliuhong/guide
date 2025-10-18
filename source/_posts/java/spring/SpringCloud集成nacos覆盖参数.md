---
title: SpringCloud集成nacos-本地覆盖远程配置
categories: 
    - Java
date: 2020-09-24 18:40:23
hide: true
tags:
	- Spring
	- SpringBoot
	- nacos
cover: https://static.jiangliuhong.top/blogimg/other/nacos_title.png 
---
# SpringCloud集成nacos-本地覆盖远程配置

> 本地覆盖远程配置即本地配置优先，常见的是使用在启动时使用`-D`配置参数

在开发阶段，为了调试开发，需要把某个配置改变，但如果直接改变配置中心的值，则会影响到其他的开发小伙伴，所以想通过在项目启动的时候，通过`-D`指定变量，例如：

```
-Dspring.application.name=test
```

但是直接这样设置是不生效的，集成`nacos`后，配置中心的配置会覆盖系统配置。

查看`spring-cloud-context`源码发现，在`org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration#insertPropertySources`方发处有判断是否覆盖系统配置

![](https://static.jiangliuhong.top/images/2021/1/1611905131604.png)

`isOverrideSystemProperties`方发的值来源于：`org.springframework.cloud.bootstrap.config.PropertySourceBootstrapProperties#overrideSystemProperties`,其默认值为true。

不难得出，我们只需要将该配置项的值设置为`false`，即可实现在启动时指定配置变量。配置如下：

```
spring:
    cloud:
        config:
            overrideSystemProperties: false
```

