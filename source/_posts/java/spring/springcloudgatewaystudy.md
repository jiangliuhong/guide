---
title: Spring Cloud Gateway 入门学习
date: 2020-07-10 23:00:55
categories: 
    - Java
    - Spring
tags:
    - Spring
    - SpringCloud
cover: https://static.jiangliuhong.top/blogimg/spring/spring.png 
---

# Spring Cloud Gateway 入门学习

> Spring Cloud Gateway 是Spring Cloud的一个项目，它是基于Spring、Webflux、SpringBoot和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效的统一的 API 路由管理方式，目标为替换 Netflix Zuul项目，其不仅提供统一的路由方式，并且基于 Filter 链的方式提供了网关基本的功能，例如：安全，监控/指标，和限流。

Spring Cloud Gateway 的主要特性有：

- 基于 Spring Framework 5，Project Reactor 和 Spring Boot 2.0
- 动态路由
- Predicates 和 Filters 作用于特定路由
- 集成 Hystrix 断路器
- 集成 Spring Cloud DiscoveryClient
- 易于编写的 Predicates 和 Filters
- 限流
- 路径重写

<font color="red">本博客示例源码地址：[https://github.com/jiangliuhong/springcloud-stu](https://github.com/jiangliuhong/springcloud-stu)</font>

## 快速启动

首先定义一个`springboot`以及`springcloud`的版本，这里我分别使用的是` 2.3.1.RELEASE`、`Hoxton.SR6`。

在pom文件定义父工程依赖：

```xml
 <parent>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-parent</artifactId>
   <version>2.3.1.RELEASE</version>
</parent>
```

在pom文件中提下如下依赖：

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

设置启动类

```java
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

添加`application.yml`文件

```yml
server:
  port: 8080
spring:
  cloud:
    gateway:
      routes:
        - id: myblog
          uri: https://jiangliuhong.top
          predicates:
            - Path=/myblog
```

各字段含义如下：

- id：我们自定义的路由 ID，保持唯一
- uri：目标服务地址
- predicates：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。

然后启动项目，通过访问`http://127.0.0.1:8080/myblog`就能访问到我的博客了,但是此时页面会有很多资源找不到，原因是页面上写死的`/xxx.css`、`/xxx.js`的资，如果要解决的话，需要在原网站上进行修改。

当然路由配置也可以在java类中配置，例如：

```java
@Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
		return builder.routes()
				.route("myblog", r -> r.path("/myblog")
						.uri("https://jiangliuhong.top"))
				.build();
	}

```

## 配和注册中心使用

在微服务中，一般我们有很多的服务，可以手动配置路由，如：

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: myblog
          uri: lb://SPRING-CLOUD-BLOG
          predicates:
            - Path=/myblog
```

可以看出，唯一的区别就是`uri`处发生了改变，其固定格式为，lb://注册中心服务名

一般注册中心中是有很多服务的，如果每个这样手动配置，那将是非常困难的，好在gateway提供了配置，可以自动根据注册中心路由，并随服务的改变而改变路由，其配置如下：

```yml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
```

然后就可以根据配置中心服务名访问，例如：http://12.0.0.1:8080/SPRING-CLOUD-BLOG,这里的`SPRING-CLOUD-BLOG`为注册中心的服务名，但是这个大写的，感觉是有点难看啊。所以gateway提供了一个配置，可以强制转为小写，其配置如下：

```yml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          lower-case-service-id: true
```

此时的访问路径为:http://127.0.0.1:8080/spring-cloud-blog

## 路由规则

> 路由规则来自http://www.ityouknow.com/springcloud/2018/12/12/spring-cloud-gateway-start.html
>
> Spring Cloud Gateway 的功能很强大，我们仅仅通过 Predicates 的设计就可以看出来，前面我们只是使用了 predicates 进行了简单的条件匹配，其实 Spring Cloud Gataway 帮我们内置了很多 Predicates 功能。
>
> Spring Cloud Gateway 是通过 Spring WebFlux 的 `HandlerMapping` 做为底层支持来匹配到转发路由，Spring Cloud Gateway 内置了很多 Predicates 工厂，这些 Predicates 工厂通过不同的 HTTP 请求参数来匹配，多个 Predicates 工厂可以组合使用。

### Gateway中的Predicate

在 Spring Cloud Gateway 中 Spring 利用 Predicate 的特性实现了各种路由匹配规则，有通过 Header、请求参数等不同的条件来进行作为条件匹配到对应的路由。网上有一张图总结了 Spring Cloud 内置的几种 Predicate 的实现。

 ![](https://static.jiangliuhong.top/blogimg/spring/spring-cloud-gateway3.png)

### 通过时间匹配

Predicate 支持设置一个时间，在请求进行转发的时候，可以通过判断在这个时间之前或者之后进行转发。比如我们现在设置只有在2019年1月1日才会转发到我的网站，在这之前不进行转发，我就可以这样配置：

```yml
spring:
  cloud:
    gateway:
      routes:
       - id: time_route
        uri: http://ityouknow.com
        predicates:
         - After=2018-01-20T06:06:06+08:00[Asia/Shanghai]
```

Spring 是通过 ZonedDateTime 来对时间进行的对比，ZonedDateTime 是 Java 8 中日期时间功能里，用于表示带时区的日期与时间信息的类，ZonedDateTime 支持通过时区来设置时间，中国的时区是：`Asia/Shanghai`。

After Route Predicate 是指在这个时间之后的请求都转发到目标地址。上面的示例是指，请求时间在 2018年1月20日6点6分6秒之后的所有请求都转发到地址`http://ityouknow.com`。`+08:00`是指时间和UTC时间相差八个小时，时间地区为`Asia/Shanghai`。

添加完路由规则之后，访问地址`http://localhost:8080`会自动转发到`http://ityouknow.com`。

Before Route Predicate 刚好相反，在某个时间之前的请求的请求都进行转发。我们把上面路由规则中的 After 改为 Before，如下：

```yml
spring:
  cloud:
    gateway:
      routes:
       - id: after_route
        uri: http://ityouknow.com
        predicates:
         - Before=2018-01-20T06:06:06+08:00[Asia/Shanghai]
```

就表示在这个时间之前可以进行路由，在这时间之后停止路由，修改完之后重启项目再次访问地址`http://localhost:8080`，页面会报 404 没有找到地址。

除过在时间之前或者之后外，Gateway 还支持限制路由请求在某一个时间段范围内，可以使用 Between Route Predicate 来实现。

```yml
spring:
  cloud:
    gateway:
      routes:
       - id: after_route
        uri: http://ityouknow.com
        predicates:
         - Between=2018-01-20T06:06:06+08:00[Asia/Shanghai], 2019-01-20T06:06:06+08:00[Asia/Shanghai]
```

这样设置就意味着在这个时间段内可以匹配到此路由，超过这个时间段范围则不会进行匹配。通过时间匹配路由的功能很酷，可以用在限时抢购的一些场景中。

### 通过 Cookie 匹配

Cookie Route Predicate 可以接收两个参数，一个是 Cookie name ,一个是正则表达式，路由规则会通过获取对应的 Cookie name 值和正则表达式去匹配，如果匹配上就会执行路由，如果没有匹配上则不执行。

```yml
spring:
  cloud:
    gateway:
      routes:
       - id: cookie_route
         uri: http://ityouknow.com
         predicates:
         - Cookie=ityouknow, kee.e
```

使用 curl 测试，命令行输入:

```
curl http://localhost:8080 --cookie "ityouknow=kee.e"
```

则会返回页面代码，如果去掉`--cookie "ityouknow=kee.e"`，后台汇报 404 错误。

### 通过 Header 属性匹配

Header Route Predicate 和 Cookie Route Predicate 一样，也是接收 2 个参数，一个 header 中属性名称和一个正则表达式，这个属性值和正则表达式匹配则执行。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: http://ityouknow.com
        predicates:
        - Header=X-Request-Id, \d+
```

使用 curl 测试，命令行输入:

```
curl http://localhost:8080  -H "X-Request-Id:666666" 
```

则返回页面代码证明匹配成功。将参数`-H "X-Request-Id:666666"`改为`-H "X-Request-Id:neo"`再次执行时返回404证明没有匹配。

### 通过 Host 匹配

Host Route Predicate 接收一组参数，一组匹配的域名列表，这个模板是一个 ant 分隔的模板，用`.`号作为分隔符。它通过参数中的主机地址作为匹配规则。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: http://ityouknow.com
        predicates:
        - Host=**.ityouknow.com
```

使用 curl 测试，命令行输入:

```
curl http://localhost:8080  -H "Host: www.ityouknow.com" 
curl http://localhost:8080  -H "Host: md.ityouknow.com" 
```

经测试以上两种 host 均可匹配到 host_route 路由，去掉 host 参数则会报 404 错误。

### 通过请求方式匹配

可以通过是 POST、GET、PUT、DELETE 等不同的请求方式来进行路由。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: http://ityouknow.com
        predicates:
        - Method=GET
```
使用 curl 测试，命令行输入:
```
# curl 默认是以 GET 的方式去请求
curl http://localhost:8080
```

测试返回页面代码，证明匹配到路由，我们再以 POST 的方式请求测试。

```
# curl 默认是以 GET 的方式去请求
curl -X POST http://localhost:8080
```

返回 404 没有找到，证明没有匹配上路由

### 通过请求路径匹配

Path Route Predicate 接收一个匹配路径的参数来判断是否走路由。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: http://ityouknow.com
        predicates:
        - Path=/foo/{segment}
```

使用 curl 测试，命令行输入:

```
curl http://localhost:8080/foo/1
curl http://localhost:8080/foo/xx
curl http://localhost:8080/boo/xx
```

经过测试第一和第二条命令可以正常获取到页面返回值，最后一个命令报404，证明路由是通过指定路由来匹配。

### 通过请求参数匹配

Query Route Predicate 支持传入两个参数，一个是属性名一个为属性值，属性值可以是正则表达式。

```
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: http://ityouknow.com
        predicates:
        - Query=smile
```

这样配置，只要请求中包含 smile 属性的参数即可匹配路由。

使用 curl 测试，命令行输入:

```
curl localhost:8080?smile=x&id=2
```

经过测试发现只要请求汇总带有 smile 参数即会匹配路由，不带 smile 参数则不会匹配。

还可以将 Query 的值以键值对的方式进行配置，这样在请求过来时会对属性值和正则进行匹配，匹配上才会走路由。

```
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: http://ityouknow.com
        predicates:
        - Query=keep, pu.
```

这样只要当请求中包含 keep 属性并且参数值是以 pu 开头的长度为三位的字符串才会进行匹配和路由。

使用 curl 测试，命令行输入:

```
curl localhost:8080?keep=pub
```

测试可以返回页面代码，将 keep 的属性值改为 pubx 再次访问就会报 404,证明路由需要匹配正则表达式才会进行路由。

### 通过请求 ip 地址进行匹配

Predicate 也支持通过设置某个 ip 区间号段的请求才会路由，RemoteAddr Route Predicate 接受 cidr 符号(IPv4 或 IPv6 )字符串的列表(最小大小为1)，例如 192.168.0.1/16 (其中 192.168.0.1 是 IP 地址，16 是子网掩码)。

```
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: http://ityouknow.com
        predicates:
        - RemoteAddr=192.168.1.1/24
```

可以将此地址设置为本机的 ip 地址进行测试。

```
curl localhost:8080
```

果请求的远程地址是 192.168.1.10，则此路由将匹配。

### 组合使用

上面为了演示各个 Predicate 的使用，我们是单个单个进行配置测试，其实可以将各种 Predicate 组合起来一起使用。

例如：

```
spring:
  cloud:
    gateway:
      routes:
       - id: host_foo_path_headers_to_httpbin
        uri: http://ityouknow.com
        predicates:
        - Host=**.foo.org
        - Path=/headers
        - Method=GET
        - Header=X-Request-Id, \d+
        - Query=foo, ba.
        - Query=baz
        - Cookie=chocolate, ch.p
        - After=2018-01-20T06:06:06+08:00[Asia/Shanghai]
```

各种 Predicates 同时存在于同一个路由时，请求必须同时满足所有的条件才被这个路由匹配。

## 常用的过滤器

> 参见官网：https://cloud.spring.io/spring-cloud-gateway/reference/html/#gatewayfilter-factories

### Gateway中的Filters

在Spring Cloud Gateway中，过滤器的作用主要为对请求与返回进行处理，可以简单的理解为一个方法的前置事件与后置事件，所以一般过滤球被分为两种，分别为“pre”与“post”。在“pre”类型的过滤器可以做参数校验、权限校验、流量监控、日志输出、协议转换等，在“post”类型的过滤器中可以做响应内容、响应头的修改，日志的输出，流量监控等。

在Gateway中有许多的过滤器，详细见Gateway官网https://cloud.spring.io/spring-cloud-gateway/reference/html/#the-addrequestparameter-gatewayfilter-factory

![filters](https://static.jiangliuhong.top/blogimg/spring/gatewayfilters.png)

### AddRequestHeader

根据过滤器名称，不难理解，该过滤器主要的作用为请求你增加一个`header`，配置方式如下：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        filters:
        - AddRequestHeader=X-Request-red, blue
```

同时也可以配合`{segment}`参数来使用，例如：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment}
        filters:
        - AddRequestHeader=X-Request-Red, Blue-{segment}
```

上面的示例中过滤器主要的作用为获取`URI`中red后面的内容，并将其放在`header`信息中。

<font color="red">注意：对于参数的使用适用于所有的过滤器</font>

### AddRequestParameter

该过滤器的作用为在请求中新增请求参数，例如：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_parameter_route
        uri: https://example.org
        filters:
        - AddRequestParameter=red, blue
```

### AddResponseHeader

该过滤器的作用为新增返回`header`信息，例如：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: add_response_header_route
        uri: https://example.org
        filters:
        - AddResponseHeader=X-Response-Red, Blue
```

### DedupeResponseHeader

该过滤器的作用为去除返回信息中重复的`header`信息，例如：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: dedupe_response_header_route
        uri: https://example.org
        filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
```

简单举个例子：

我们在Gateway以及微服务上都设置了CORS（解决跨域）header，如果不做任何配置，请求 -> 网关 -> 微服务，获得的响应就是这样的：

```
Access-Control-Allow-Credentials: true, true
Access-Control-Allow-Origin: https://musk.mars, https://musk.mars
```

也就是Header重复了。要想把这两个Header去重，只需设置成如下即可。

```
filters:
- DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
```

也就是说，想要去重的Header如果有多个，用空格分隔即可；

去重策略：

- RETAIN_FIRST: 默认值，保留第一个值
- RETAIN_LAST: 保留最后一个值
- RETAIN_UNIQUE: 保留所有唯一值，以它们第一次出现的顺序保留

### CircuitBreaker

该过滤器需要配合底层断路器使用，例如:`Hystrix`，主要作用为对服务进行熔断处理，目前`Spring Cloud CircuitBreaker`主要提供了四种断路器的支持，分别为：

- Netfix Hystrix
- Resilience4J
- Sentinel
- Spring Retry

关于熔断过滤器可参考：https://blog.csdn.net/netyeaxi/article/details/104469248

### FallbackHeaders

该过滤器也是对熔断器的支持，详细可参看官网：https://cloud.spring.io/spring-cloud-gateway/reference/html/#fallback-headers

### MapRequestHeader

该过滤器作用与`AddRequestHeader`相同，都是新增请求`header`，区别是改过滤器会判断其配置的`header`是否存在，配置方式为：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: map_request_header_route
        uri: https://example.org
        filters:
        - MapRequestHeader=Blue, X-Request-Red
```

### PrefixPath

该过滤器的作用为匹配的路由添加前缀。例如：访问`${GATEWAY_URL}/hello` 会转发到`https://example.org/mypath/hello` ，配置方式为：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - PrefixPath=/mypath
```

### PreserveHostHeader

如果不配置该过滤器，那么名为`HOST`的`header`将有`HttpClient`控制，如果配置该过滤器，那么会设置一个请求属性`preserveHostHeader=true`，路由过滤器会检查从而去判断是否要发送原始的、名为Host的Header。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: preserve_host_route
        uri: https://example.org
        filters:
        - PreserveHostHeader
```

### RedirectTo

顾名思义，配置该过滤器后会对请求进行重定向，让问 `${GATEWAY_URL}/hello` 会重定向到 `https://ecme.org/hello` ，并且携带一个 `Location:https://acme.org` 的Header。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - RedirectTo=302, https://acme.org
```

### RemoveRequestHeader

移除请求`header`

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: removerequestheader_route
        uri: https://example.org
        filters:
        - RemoveRequestHeader=X-Request-Foo
```

### RemoveResponseHeader

移除响应`header`

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: removeresponseheader_route
        uri: https://example.org
        filters:
        - RemoveResponseHeader=X-Response-Foo
```

### RemoveRequestParameter

移除请求参数

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: removerequestparameter_route
        uri: https://example.org
        filters:
        - RemoveRequestParameter=red
```

### RewritePath

该过滤器的作用主要为重写请求，某些情况下，我们通过网关路由到一个地址，会加上路由前缀，通过该过滤器可以实现问访问`red/blue`会将路径改为`blue`再转发。需要注意的是，由于YAML语法，需用`$\` 替换 `$` 。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://example.org
        predicates:
        - Path=/red/**
        filters:
        - RewritePath=/red(?<segment>/?.*), $\{segment}
```

