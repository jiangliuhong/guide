---
title: Spring Cloud Gateway动态路由实现
categories: 
    - Java
    - Spring
date: 2020-07-10 23:10:55
comments: valine
tags:
	- Spring
cover: https://static.jiangliuhong.top/blogimg/spring/spring.png
---

# Spring Cloud Gateway动态路由实现

在实现动态路由之前，你至少能搭建一个简单的`Gateway`项目，并对其有一定的了解，具体可以参考[Spring Cloud Gateway 入门学习](/2020/07/10/java/spring/springcloudgatewaystudy/)

本文要实现的动态路由为在数据库中存储路由配置，并提供一个页面可以配置路由。

## 准备工作

数据库脚本（以下脚本针对`PostgreSQL`数据库，脚本源文件地址）：

```sql
-- 创建模式
create schema db_base;
-- 创建表
create table db_base.t_route(
	c_id varchar(300) primary key not null,
	c_name varchar(300) not null,
	c_predicates varchar(1200),
	c_url varchar(900) not null,
	c_filters varchar(1200)
);
-- 初始化一条测试数据
INSERT INTO "db_base"."t_route"("c_id", "c_name", "c_predicates", "c_url", "c_filters") VALUES ('scs-blog', '我的博客', '["Path=/blog/**"]', 'https://jiangliuhong.top/', '["RewritePath=/blog(?<segment>/?.*), $\\{segment}"]');
```

在`Gateway`中追加以下依赖：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid-spring-boot-starter</artifactId>
</dependency>
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
</dependency>
```

修改`application.yml`文件：

```yml
spring:
  datasource:
    url: jdbc:postgresql://127.0.0.1:5432/scs
    username: sa
    password: 6789@jkl
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: org.postgresql.Driver
    druid:
      initial-size: 5
      min-idle: 5
      max-active: 20
      validation-query: select version()
  jpa:
    database: postgresql
    show-sql: true
    properties:
      hibernate:
        hbm2ddl:
          # 配置开启自动更新表结构
          auto: update
```

## 核心方法

> Gateway源码地址：https://github.com/jiangliuhong/springcloud-stu/tree/master/scs-gateway

### RouteService

> 原文件地址：https://github.com/jiangliuhong/springcloud-stu/blob/master/scs-gateway/src/main/java/top/jiangliuhong/scs/gateway/service/RouteService.java

```java
@Service
public class RouteService {

    private static final Logger logger = LoggerFactory.getLogger(RouteService.class);

    @Resource
    private RouteRepository routeRepository;

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    /**
     * 加载配置
     */
    public void loadRoute() {
        // getRoutes 从数据库加载数据
        List<Route> routes = getRoutes();
        if (CollectionUtils.isEmpty(routes)) {
            logger.warn("当前没有路由需要加载");
            return;
        }
        for (Route route : routes) {
            RouteDefinition definition = new RouteDefinition();
            definition.setId(route.getId());
            try {
                definition.setUri(new URI(route.getUrl()));
            } catch (URISyntaxException e) {
                logger.error("初始化uri失败:" + route.getUrl(), e);
                continue;
            }
            if (StringUtils.isBlank(route.getPredicates())) {
                logger.warn("{}[{}]的Predicates为空", route.getId(), route.getName());
                continue;
            }
            // 路由设置
            List<PredicateDefinition> predicates = Lists.newArrayList();
            List<String> predicatesList = JSON.parseArray(route.getPredicates(), String.class);
            for (String p : predicatesList) {
                predicates.add(new PredicateDefinition(p));
            }
            definition.setPredicates(predicates);
            // 过滤器设置
            if (StringUtils.isNotBlank(route.getFilters())) {
                List<FilterDefinition> filters = Lists.newArrayList();
                List<String> filtersList = JSON.parseArray(route.getFilters(), String.class);
                for (String f : filtersList) {
                    filters.add(new FilterDefinition(f));
                }
                definition.setFilters(filters);
            }
          	// 保存路由信息
            routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        }
        // 刷新路由
        ApplicationEventUtils.pushEvent(new RefreshRoutesEvent(this));
    }
}
```

###  RouteRunner

> 启动时加载数据库路由

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RouteRunner implements CommandLineRunner {
    @Autowired
    private RouteService routeService;

    @Override
    public void run(String... args) throws Exception {
        routeService.loadRoute();
    }
}
```

## 前端页面

> 前端源码：https://github.com/jiangliuhong/springcloud-stu/tree/master/scs-websource/scsweb-gateway

![](https://static.jiangliuhong.top/blogimg/spring/gatewayconsolepreview.png)

