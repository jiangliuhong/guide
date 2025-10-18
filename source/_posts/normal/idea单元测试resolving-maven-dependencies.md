---
title: idea单元测试resolving maven dependencies
date: 2020-04-09 18:40:45
categories:
  - 随笔
tags:
cover:
---

最近在springboot项目中进行单元测试时，遇见这样一个问题，显示卡在这样一个画面：

![](https://static.jiangliuhong.top/blogimg/other/idea-test-junit-platform.png)

然后出现这样一个错误：

```
Error running 'AuditApplicationTests.contextLoads': Failed to resolve org.junit.platform:junit-platform-launcher:1.5.2
```

springboot版本：2.2.6.RELEASE

idea版本：2019.3.4

解决办法：在项目的`pom.xml`文件中加入如下依赖

```
		<dependency>
			<groupId>org.junit.platform</groupId>
			<artifactId>junit-platform-launcher</artifactId>
			<scope>test</scope>
		</dependency>
```