---
title: SpringMvc自定义参数解析与返回值处理
categories: 
    - Java
    - Spring
date: 2018-10-18 21:52:08
tags:
    - Spring
cover: https://static.jiangliuhong.top/blogimg/spring/spring.png   
---
# SpringMvc自定义参数解析与返回值处理

> 近日在做项目的时候，需要解析客户端传来的经过`AES`加密处理的实体信息，同时也需要向客户端返回经过`AES`加密的实体信息，在项目初期，都是在`Controller`方法中去调用某个工具类进行decode、encode操作比较繁琐，于是去寻求解决办法，在翻阅了`SpringMvc`解析参数的源码后，仿照`@RequestBody`的进行以下实现。本文基于`SpringBoot 2.0`即`SpringMvc 5.0.6`。

## SpringMvc 参数绑定原理

### ArgumentResolver与ReturnValueHandler

通过在maven在项目中引入SpringMvc依赖，你可以使用ide的快捷键（比如，idea是ctrl+n)查找类RequestBody，其类注释如下：

```java
/**
 * Annotation indicating a method parameter should be bound to the body of the web request.
 * The body of the request is passed through an {@link HttpMessageConverter} to resolve the
 * method argument depending on the content type of the request. Optionally, automatic
 * validation can be applied by annotating the argument with {@code @Valid}.
 *
 * <p>Supported for annotated handler methods in Servlet environments.
 *
 * @author Arjen Poutsma
 * @since 3.0
 * @see RequestHeader
 * @see ResponseBody
 * @see org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
 */
```

接着继续查找类注释中的类`RequestMappingHandlerAdapter`，查看其源码可以发现，其源码的注释中有这样两行代码：

```java
 * @see HandlerMethodArgumentResolver
 * @see HandlerMethodReturnValueHandler
```

分别查看这两个类:

### HandlerMethodArgumentResolver

> 在给定请求的上下文中，将方法参数解析为参数值的策略接口。

HandlerMethodArgumentResolver中有两个接口

```java 
//判断传入的参数是否被该方法所支持
boolean supportsParameter(MethodParameter parameter);
//参数解析方法
Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
```

其中resolveArgument方法的返回值，会被作为参数传入到Controller的方法参数中。

### HandlerMethodReturnValueHandler

> 处理程序方法返回值的策略接口。

同样的，HandlerMethodReturnValueHandler中也有两个接口

```java
//判断传入的参数是否被该方法所支持
boolean supportsReturnType(MethodParameter returnType);
//返回值解析
void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;
```

在handleReturnValue我们可以直接构造返回给客户端的内容。

### MethodParameter

按照官方的说明，这个类封装了方法参数的规范，记录了一个方法的类注解、方法注解、方法参数。

我们在`supportsReturnType`方法去判断这个类是否拥有指定注解(自定义注解)，从而进行相应的处理逻辑，另外我们还可以通过这个类的对象去获取Controller方法的参数类型，比如：

```java
parameter.getNestedGenericParameterType()
```

如果该注解在方法的参数上，即`ElementType.PARAMETER`，例子见下方，则getNestedGenericParameterType方法返回的为其制定参数的类型：

```java
@Controller
public class TestApi extends BaseController {
    private static Logger logger = LoggerFactory.getLogger(TestApi.class);
    @PostMapping(value = "/encryptTest")
    public UserDo encryptTest(@EncryptBody UserDo user) {
        return user;
    }
}
```

如果将注解打在方法上，即`ElementType.METHOD`，例子见下方，则getNestedGenericParameterType方法返回的为方法返回值类型：

```java
@EncryptBody
public UserDo encryptTest(){
}
```

### NativeWebRequest

NativeWebRequest是WebRequest接口的扩展 ，是springmvc专门定义用来供框架内部使用，特别是通用参数解析代码。

在ArgumentResolver与ReturnValueHandler中，我们可以使用其获取`HttpServletRequest`与`HttpServletResponse`，从而实现对参数的解析与返回值的构建。

```java
HttpServletRequest servletRequest = nativeWebRequest.getNativeRequest(HttpServletRequest.class);
HttpServletResponse response = nativeWebRequest.getNativeResponse(HttpServletResponse.class);
```

### RequestMappingHandlerAdapter

继续查看`RequestMappingHandlerAdapter`的源码，分别以下四个变量：

- customArgumentResolvers：自定义的参数解析器
- argumentResolvers：默认的参数解析器，通过`getDefaultArgumentResolvers`方法可查看其具体的初始换方式。
- customReturnValueHandlers：自定义的返回值处理器
- returnValueHandlers：默认的返回值处理器，通过`getDefaultReturnValueHandlers`方法可查看其具体的初始化方式

getDefaultArgumentResolvers源码如下：

```java
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		resolvers.add(new ModelMethodProcessor());
		resolvers.add(new MapMethodProcessor());
		resolvers.add(new ErrorsMethodArgumentResolver());
		resolvers.add(new SessionStatusMethodArgumentResolver());
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}
```

getDefaultReturnValueHandlers源码如下：

```java 
private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>();

		// Single-purpose return value types
		handlers.add(new ModelAndViewMethodReturnValueHandler());
		handlers.add(new ModelMethodProcessor());
		handlers.add(new ViewMethodReturnValueHandler());
		handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters(),
				this.reactiveAdapterRegistry, this.taskExecutor, this.contentNegotiationManager));
		handlers.add(new StreamingResponseBodyReturnValueHandler());
		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));
		handlers.add(new HttpHeadersReturnValueHandler());
		handlers.add(new CallableMethodReturnValueHandler());
		handlers.add(new DeferredResultMethodReturnValueHandler());
		handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));

		// Annotation-based return value types
		handlers.add(new ModelAttributeMethodProcessor(false));
		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));

		// Multi-purpose return value types
		handlers.add(new ViewNameMethodReturnValueHandler());
		handlers.add(new MapMethodProcessor());

		// Custom return value types
		if (getCustomReturnValueHandlers() != null) {
			handlers.addAll(getCustomReturnValueHandlers());
		}

		// Catch-all
		if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
			handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
		}
		else {
			handlers.add(new ModelAttributeMethodProcessor(true));
		}

		return handlers;
	}
```

## 自定义解析器与构造器

> 在自定义的过程中依赖了项目中的一些工具类，比如：`AbstractEcryptMappingHadler`、`Encrypt`等等，由于项目中预留了多种加密方式的接口，类稍有点过多，此处就不一一贴出，如有需要，请移驾github查看源码[https://github.com/jiangliuhong/RedisWClient-server](https://github.com/jiangliuhong/RedisWClient-server)查看源码(包路径为：pers.jarome.redis.wclient.common.web.encrypt)，当然，你也可以移除这些依赖，然后自定义逻辑。

### 自定义EncryptArgumentResolver

```java
package pers.jarome.redis.wclient.common.web.encrypt.method.resolver;

import org.springframework.core.MethodParameter;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;
import pers.jarome.redis.wclient.common.web.encrypt.anno.EncryptBody;
import pers.jarome.redis.wclient.common.web.encrypt.constants.EncryptMethod;
import pers.jarome.redis.wclient.common.web.encrypt.entity.Encrypt;
import pers.jarome.redis.wclient.common.web.encrypt.exception.EncryptException;
import pers.jarome.redis.wclient.common.web.encrypt.method.AbstractEcryptMappingHadler;

import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

/**
 * EncryptArgumentResolver
 *
 * @author jiangliuhong
 * @description 加密解析器
 * @date 2018/8/17 9:35
 */
public class EncryptArgumentResolver extends AbstractEcryptMappingHadler implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return hasEncryptAnnotaion(parameter);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        //请自定义你的处理逻辑。
        //你可以根据HttpServletRequest去获取你想要的
        //此处我是通过webRequest获取body中的内容，然后将其进行AES解密，然后转为controller方法中需要的对象
        String body = getRequestBody(webRequest);
        EncryptBody encryptBody = parameter.getAnnotatedElement().getAnnotation(EncryptBody.class);
        Encrypt encrypt = getEncrypt(encryptBody.method());
        if(encrypt == null){
            throw new EncryptException("Not Found Encrypt.");
        }
        return encrypt.decode(body, parameter.getNestedGenericParameterType());

    }

    private Boolean hasEncryptAnnotaion(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(EncryptBody.class);
    }

    private String getRequestBody(NativeWebRequest webRequest) throws IOException {
        HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
        ServletInputStream inputStream = servletRequest.getInputStream();
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
        StringBuilder body = new StringBuilder();
        String str;
        while ((str = bufferedReader.readLine()) != null) {
            body.append(str);
        }
        return body.toString();
    }
}

```

### 自定义EncryptBodyRturnValueHandler

```java
package pers.jarome.redis.wclient.common.web.encrypt.method.handler;

import com.alibaba.fastjson.JSON;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodReturnValueHandler;
import org.springframework.web.method.support.ModelAndViewContainer;
import pers.jarome.redis.wclient.common.web.encrypt.anno.EncryptBody;
import pers.jarome.redis.wclient.common.web.encrypt.entity.Encrypt;
import pers.jarome.redis.wclient.common.web.encrypt.method.AbstractEcryptMappingHadler;

import javax.servlet.http.HttpServletResponse;
import java.io.PrintWriter;

/**
 * 
 * EncryptBodyRturnValueHandler
 * @description 加密实体返回值组装器
 * @author jiangliuhong
 * @date 2018/8/16 21:53
 */
public class EncryptBodyRturnValueHandler extends AbstractEcryptMappingHadler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        return returnType.hasMethodAnnotation(EncryptBody.class);
    }

    @Override
    public void handleReturnValue(Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        //请自定义你的处理逻辑。
        //此处我是将返回值进行AES加密，然后返回给客户端
        EncryptBody encryptBody = returnType.getMethodAnnotation(EncryptBody.class);
        mavContainer.setRequestHandled(true);
        HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
        response.setContentType(MediaType.APPLICATION_JSON_UTF8_VALUE);
        PrintWriter outWriter = response.getWriter();
        Encrypt encrypt = getEncrypt(encryptBody.method());
        Object encode = encrypt.encode(returnValue);
        String jsonString = "";
        if(encode!=null) {
            jsonString = JSON.toJSONString(encode);
        }
        outWriter.write(jsonString);
        outWriter.flush();
        outWriter.close();
    }
}

```

### 注册

继续调试跟踪代码，发现在`WebMvcConfigurationSupport`类(只有高版本的SpringBoot才具有该类，低版本的为`WebMvcConfigurerAdapter`该类的逻辑，由于没有深入查看，此处就不做赘述了)中有这样一个方法：

```java
@Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
        //初始化一个RequestMappingHandlerAdapter
		RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
		adapter.setContentNegotiationManager(mvcContentNegotiationManager());
		adapter.setMessageConverters(getMessageConverters());
		adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer());
		adapter.setCustomArgumentResolvers(getArgumentResolvers());
		adapter.setCustomReturnValueHandlers(getReturnValueHandlers());

		if (jackson2Present) {
			adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
			adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
		}

		AsyncSupportConfigurer configurer = new AsyncSupportConfigurer();
		configureAsyncSupport(configurer);
		if (configurer.getTaskExecutor() != null) {
			adapter.setTaskExecutor(configurer.getTaskExecutor());
		}
		if (configurer.getTimeout() != null) {
			adapter.setAsyncRequestTimeout(configurer.getTimeout());
		}
		adapter.setCallableInterceptors(configurer.getCallableInterceptors());
		adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

		return adapter;
	}
```

该方法的作用为在系统初始化的时候，返回系统以及用户自定义的解析器等。

从上诉代码不难看出，在加载用户自定义的处理器的代码为：

```java
adapter.setCustomArgumentResolvers(getArgumentResolvers());
adapter.setCustomReturnValueHandlers(getReturnValueHandlers());
```

刨根究底，发现`getArgumentResolvers`与`getReturnValueHandlers`的数据源源为`WebMvcConfigurer`接口中的`addArgumentResolvers`与`addReturnValueHandlers`。此时我们可以通过自定义类，实现这两个接口来实现注册了。

#### SpringBoot1.x注册方法

```java
package pers.jarome.redis.wclient.app.config;

import org.springframework.stereotype.Component;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.HandlerMethodReturnValueHandler;
import org.springframework.web.servlet.config.annotation.InterceptorRegistration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import pers.jarome.redis.wclient.common.web.encrypt.method.handler.EncryptBodyRturnValueHandler;
import pers.jarome.redis.wclient.common.web.interceptor.AuthenticationInterceptor;
import pers.jarome.redis.wclient.common.web.encrypt.method.resolver.EncryptArgumentResolver;

import java.util.List;

/**
 * Web环境配置
 *
 * @author jiangliuhong
 * @date 2017/12/29
 **/
@Component
public class WebMvcConfigAdapter extends WebMvcConfigurerAdapter {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(new EncryptArgumentResolver());
    }
    @Override
    public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> returnValueHandlers) {
        returnValueHandlers.add(new EncryptBodyRturnValueHandler());
    }
}

```

#### SpringBoot2.0注册方法

如果你的SpringBoot版本大于等于2.0，那么WebMvcConfigurerAdapter过期，此时应该使用新的`WebMvcConfigurationSupport`

```java
package pers.jarome.redis.wclient.app.config;

import org.springframework.stereotype.Component;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.HandlerMethodReturnValueHandler;
import org.springframework.web.servlet.config.annotation.InterceptorRegistration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;
import pers.jarome.redis.wclient.common.web.encrypt.method.handler.EncryptBodyRturnValueHandler;
import pers.jarome.redis.wclient.common.web.encrypt.method.resolver.EncryptArgumentResolver;
import pers.jarome.redis.wclient.common.web.interceptor.AuthenticationInterceptor;

import java.util.List;

/**
 * 
 * WebMvcConfig
 * @description Web环境配置
 * @author jiangliuhong
 * @date 2018/8/19 0:46
 */
@Component
public class WebMvcConfig extends WebMvcConfigurationSupport {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        InterceptorRegistration ir = registry.addInterceptor(new AuthenticationInterceptor());
        ir.addPathPatterns("/**");
    }

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(new EncryptArgumentResolver());
    }

    @Override
    public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> returnValueHandlers) {
        returnValueHandlers.add(new EncryptBodyRturnValueHandler());
    }

}
```

#### 非SpringBoot注册方法

> 对于非SpringBoot项目的注册方式大同小异，也就是xml配置与类配置的区别。

对于非SpringBoot的项目，在其springmvc的配置文件中加入以下代码即可：

```xml
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
        <!-- 自定义参数解析器 -->
        <property name="customArgumentResolvers">
            <list>
                <bean class="pers.jarome.redis.wclient.common.web.encrypt.method.resolver.EncryptArgumentResolver" />
            </list>
        </property>
    <property name="customReturnValueHandlers">
            <list>
                <bean class="pers.jarome.redis.wclient.common.web.encrypt.method.handler.EncryptBodyRturnValueHandler" />
            </list>
        </property>
    </bean>
```

## 结语

在自定义解析器时，我是基于`RequestMappingHandlerAdapter`进行封装实现的，在这个类中通过加入自定义的Resolver与Handler，从而达到我们期望的参数绑定与返回值处理效果，另外，我们还可以定义messageConverters等，当然，也可以自定义一个HandlerAdapter。最后再引入一个类`WebMvcAutoConfiguration`作为下一次Spring源码学习的主题吧。
