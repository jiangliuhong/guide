---
title: SpringCloud快速构建
categories: 
    - Java
    - Spring
date: 2019-05-22 09:55:08
tags:
    - Spring
    - SpringCloud
    - 网关
cover: https://static.jiangliuhong.top/blogimg/spring/springcloud.jpg
---
# SpringCloud快速构建

> SpringCloud是2014年底Spring团队基于SpringBoot开发的，推出的Java领域微服务架构完整解决方案。主要包括服务注册于发现、配置中心、全链路监控、API网关、熔断器等选型中立的开源组件。

基础组件列表如下：

| 名称    | 功能             | 简介               |
| ------- | ---------------- | ------------------ |
| Eureka  | 注册中心         | 保证一致性与高可用 |
| Consul  | 注册中心         | 保证强一致性       |
| Zuul    | 网关             | 第一代网关         |
| Gateway | 网关             | 第二代网关         |
| Ribbon  | 负载均衡         | 进程内负载均衡     |
| Hystrix | 熔断器           | 延迟、容错         |
| Fegin   | 声明式HTTP客户端 |                    |
| Sleuth  | 链路追踪         |                    |
| Config  | 配置中心         |                    |
| Bus     | 总线             |                    |

## Eureka

Eureka是Netflix开源的一款提供服务注册和发现的产品，它提供了完整的Service Registry和Service Discovery实现。也是springcloud体系中最重要最核心的组件之一。

添加依赖：

```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-eureka-server</artifactId>
	</dependency>
</dependencies>
```

在启动类中添加`@EnableEurekaServer`注解：

```java
@SpringBootApplication
@EnableEurekaServer
public class SpringCloudEurekaApplication {
	public static void main(String[] args) {
		SpringApplication.run(SpringCloudEurekaApplication.class, args);
	}
}
```

编写对应的配置文件：

```yaml
spring:
	application:
		name: spring-cloud-eureka

server:
	port: 8000
eureka:
	client:
		register-with-eureka: false
		fetch-registry: false
	serviceUrl:
		defaultZone: http://localhost:${server.port}/eureka/
```

## Feign

Spring Cloud Feign是一套基于Netflix Feign实现的声明式服务调用客户端。它使得编写Web服务客户端变得更加简单。我们只需要通过创建接口并用注解来配置它既可完成对Web服务接口的绑定。它具备可插拔的注解支持，包括Feign注解、JAX-RS注解。它也支持可插拔的编码器和解码器。Spring Cloud Feign还扩展了对Spring MVC注解的支持，同时还整合了Ribbon和Eureka来提供均衡负载的HTTP客户端实现。

引入依赖：

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

在启动类中添加`@EnableFeignClients`注解

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class Application {

	public static void main(String[] args) {
		new SpringApplicationBuilder(Application.class).web(true).run(args);
	}
}
```

创建一个Feign的客户端接口定义。使用`@FeignClient`注解来指定这个接口所要调用的服务名称，接口中定义的各个函数使用Spring MVC的注解就可以来绑定服务提供方的REST接口，比如下面就是绑定`eureka-client`服务的`/dc`接口的例子：

```java
@FeignClient("eureka-client")
public interface DcClient {

    @GetMapping("/dc")
    String consumer();

}
```

`@FeignClient`注解：

- name：指定FeignClient的名称，如果项目使用了Ribbon，name属性会作为微服务的名称，用于服务发现

- url: url一般用于调试，可以手动指定@FeignClient调用的地址

- decode404:当发生http 404错误时，如果该字段位true，会调用decoder进行解码，否则抛出FeignException

- configuration: Feign配置类，可以自定义Feign的Encoder、Decoder、LogLevel、Contract

- fallback: 定义容错的处理类，当调用远程接口失败或超时时，会调用对应接口的容错逻辑，fallback指定的类必须实现@FeignClient标记的接口

- fallbackFactory: 工厂类，用于生成fallback类示例，通过这个属性我们可以实现每个接口通用的容错逻辑，减少重复的代码

- path: 定义当前FeignClient的统一前缀

## Hystix

`Hystix`是一个供分布式系统使用，提供延迟和容错功能，保证复杂的分布系统在面临不可避免的失败时，仍能有其弹性。

添加依赖：

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
```

启动类上增加Hystrix的注解：

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public class HystrixRibbonApp {
    public static void main(String[] args) {
        SpringApplication.run(HystrixRibbonApp.class, args);
    }
    
    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

修改代码逻辑，在需要熔断的方法上增加@HystrixCommand注解，当调用有问题的时候就会使用fallbackMethod参数指定的方法进行服务降级：

```java
@RestController
public class HelloController {
    @Autowired
    HystrixRibbonService helloService;
    
    @RequestMapping("/hi")
    public String hello(){
        return helloService.helloService("姓名");
    }
}
@Service
public class HystrixRibbonService {
    private static final String SERVICE_NAME = "EUREKACLIENT";
    
    @Autowired
    RestTemplate restTemplate;
    
    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @HystrixCommand(fallbackMethod = "helloServiceFallBack")
    public String helloService(String name) {
        ServiceInstance serviceInstance = this.loadBalancerClient.choose(SERVICE_NAME);
        System.out.println("服务主机：" + serviceInstance.getHost());
        System.out.println("服务端口：" + serviceInstance.getPort());
        
        //  通过服务名来访问
        return restTemplate.getForObject("http://" + SERVICE_NAME + "/hello?name="+name,String.class);
    }

    @SuppressWarnings("unused")
    private String helloServiceFallBack(String name) {
        return "这个是失败的信息！";
    }
}
```

在`Feign`中同样可以使用`Hystix`，在`Feign`中指定`fallback`，示例如下：

```java
@FeignClient(value="EUREKACLIENT", fallback = HystrixFeignServiceFallback.class)
public interface HystrixFeignService {
    @RequestMapping(value = "/hello",method = RequestMethod.GET) 
    String sayHiUseFeign(@RequestParam(value = "name") String name);
}
@Component
public class HystrixFeignServiceFallback implements HystrixFeignService{
    @Override
    public String sayHiUseFeign(String name) {
        return "feign调用错误！";
    }
}
```

## Zuul

在说Zuul之前，应先理解API Gateway(API网关)的概念，API网关即给用户规定一个统一的入口，接收到用户请求后，网关在内部会分发到各个对应的服务服务上。API网关的好处：

- 简化客户端调用复杂度
- 数据裁剪以及聚合
- 多渠道支持
- 遗留系统的微服务化改造

Spring Cloud Zuul路由是微服务架构的不可或缺的一部分，提供动态路由，监控，弹性，安全等的边缘服务。Zuul是Netflix出品的一个基于JVM路由和服务端的负载均衡器。

引用依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```

配置文件:

```yaml
spring:
	application:
		name: gateway-service-zuul
server:
	port: 8888

#这里的配置表示，访问/goo/** 直接重定向到http://www.google.com
zuul:
	routes:
		baidu:
			path: /goo/**
			url: http://www.google.com
```

启动类：

```java
@SpringBootApplication
@EnableZuulProxy
public class GatewayServiceZuulApplication {

	public static void main(String[] args) {
		SpringApplication.run(GatewayServiceZuulApplication.class, args);
	}
}
```

### 网关服务化

添加eureka依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

配置文件：

```yaml
spring:
	application:
		name: gateway-service-zuul
server:
	port: 8888

zuul:
	routes:
		server-a:		
			path: /goo/**
			serviceId: server-a

eureka:
	client:
		serviceUrl:
			defaultZone: http://localhost:8000/eureka/
```

其中`server-a`为`eureka`中的服务。

## Sleuth

Sleuth是Spring Cloud的组成部分之一，为SpringCloud应用实现了一种分布式追踪解决方案，其兼容了Zipkin, HTrace和log-based追踪

几个基本术语：

- Span：基本工作单元，发送一个远程调度任务 就会产生一个Span，Span是一个64位ID唯一标识的，Trace是用另一个64位ID唯一标识的，Span还有其他数据信息，比如摘要、时间戳事件、Span的ID、以及进度ID。
- Trace：一系列Span组成的一个树状结构。请求一个微服务系统的API接口，这个API接口，需要调用多个微服务，调用每个微服务都会产生一个新的Span，所有由这个请求产生的Span组成了这个Trace。
- Annotation：用来及时记录一个事件的，一些核心注解用来定义一个请求的开始和结束 。这些注解包括以下：
  - cs - Client Sent -客户端发送一个请求，这个注解描述了这个Span的开始
  - sr-Server Received -服务端获得请求并准备开始处理它，如果将其sr减去cs时间戳便可得到网络传输的时间。
  - ss - Server Sent （服务端发送响应）–该注解表明请求处理的完成(当请求返回客户端)，如果ss的时间戳减去sr时间戳，就可以得到服务器请求的时间。
  - cr - Client Received （客户端接收响应）-此时Span的结束，如果cr的时间戳减去cs时间戳便可以得到整个请求所消耗的时间。

### Spring Cloud Sleuth和Zipkin分布式链路跟踪

#### Zipkin服务端构建

Zipkin 是一个开放源代码分布式的跟踪系统，由Twitter公司开源，它致力于收集服务的定时数据，以解决微服务架构中的延迟问题，包括数据的收集、存储、查找和展现。

每个服务向zipkin报告计时数据，zipkin会根据调用关系通过Zipkin UI生成依赖关系图，显示了多少跟踪请求通过每个服务，该系统让开发者可通过一个 Web 前端轻松的收集和分析数据，例如用户每次请求服务的处理时间等，可方便的监测系统中存在的瓶颈。

Zipkin提供了可插拔数据存储方式：In-Memory、MySql、Cassandra以及Elasticsearch。接下来的测试为方便直接采用In-Memory方式进行存储，生产推荐Elasticsearch。

首先在项目中添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.java</groupId>
        <artifactId>zipkin-server</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.java</groupId>
        <artifactId>zipkin-autoconfigure-ui</artifactId>
    </dependency>
</dependencies>
```

编写对应的启动类，使用了`@EnableZipkinServer`注解，启用Zipkin服务。

```java
@SpringBootApplication
@EnableEurekaClient
@EnableZipkinServer
public class ZipkinApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZipkinApplication.class, args);
    }
}
```

修改配置文件

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://127.0.0.1:8761/eureka/
server:
  port: 9000
spring:
  application:
    name: zipkin-server
```

配置完成后依次启动示例项目：`spring-cloud-eureka`、`zipkin-server`项目。刚问地址:`http://localhost:9000/zipkin/`可以看到Zipkin后台页面

![](https://static.jiangliuhong.top/blogimg/spring/tracing3.png)

### 客户端添加zipkin支持

在项目中添加如下依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

配置文件中添加如下代码：

```yaml
spring:
  zipkin:
    base-url: http://localhost:9000
  sleuth:
    sampler:
      percentage: 1.0
```

spring.zipkin.base-url指定了Zipkin服务器的地址，spring.sleuth.sampler.percentage将采样比例设置为1.0，也就是全部都需要。

Spring应用在监测到Java依赖包中有sleuth和zipkin后，会自动在RestTemplate的调用过程中向HTTP请求注入追踪信息，并向Zipkin Server发送这些信息。

