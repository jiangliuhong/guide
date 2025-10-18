---
title: Java调试体系总结
date: 2020-04-05 21:33:48
categories:
    - Java
tags:
    - tools
cover: https://static.jiangliuhong.top/blogimg/java/java-debugger.png
---

> 本文为`JPDA`、`本地调试`、`远程调试`的总结内容

## JPDA

> `JPDA`是`Java SE`1.2.2版本推出的Java凭条调试体系结构工具集，而从`JDK1.3.x`开始，JavaSDK就提供了对Java平台调试体系结构的直接支持。这个体系为开发人员提供了一整套用于调试 Java 程序的 API，是一套用于开发 Java 调试工具的接口和协议。

### JADA组成模块

`JPDA`定义了一个完整独立的体系，它主要分为三个独立的层次，分别为：Java虚拟机工具接口(`JVMTI`)、Java调试线协议(`JDWP`)以及Java调试接口(`JDI`)。其中，我们可以把`JVMTI`理解为调试者(`debugger`)，把`JDI`理解为被调试者`debuggee`，对用的层次见下图：

![JPDA模块层次](https://static.jiangliuhong.top/blogimg/java/JPDA%E6%A8%A1%E5%9D%97%E5%B1%82%E6%AC%A1.jpg)

通过该图，我们不难看出，`JDWP`是调试者与被调试者之间的桥梁，两者之间的交互都是通过`JDWP`进行传输。

### Java 虚拟机工具接口（JVMTI）

JDWP（Java Debug Wire Protocol）是一个为 Java 调试而设计的一个通讯交互协议，它定义了调试器和被调试程序之间传递的信息的格式。

它是一套由Java虚拟机直接提供的`native`接口，它处于整个`JPDA`体系的最底层，所有调试功能本质上都需要通过`JVMTI`来提供。同时我们开发人员还可以查看它们运行的状态，设置回调函数，控制某些环境变量，从而优化程序性能。

### Java 调试线协议（JDWP）

JDWP（Java Debug Wire Protocol）是一个为 Java 调试而设计的一个通讯交互协议，它定义了调试器和被调试程序之间传递的信息的格式。在 JPDA 体系中，作为前端（front-end）的调试者（debugger）进程和后端（back-end）的被调试程序（debuggee）进程之间的交互数据的格式就是由 JDWP 来描述的，它详细完整地定义了请求命令、回应数据和错误代码，保证了前端和后端的 JVMTI 和 JDI 的通信通畅。

### Java 调试接口（JDI）

JDI（Java Debug Interface）是三个模块中最高层的接口，在多数的 JDK 中，它是由 Java 语言实现的。

JDI 由针对前端定义的接口组成，通过它，调试工具开发人员就能通过前端虚拟机上的调试器来远程操控后端虚拟机上被调试程序的运行，JDI 不仅能帮助开发人员格式化 JDWP 数据，而且还能为 JDWP 数据传输提供队列、缓存等优化服务。从理论上说，开发人员只需使用 JDWP 和 JVMTI 即可支持跨平台的远程调试，但是直接编写 JDWP 程序费时费力，而且效率不高。因此基于 Java 的 JDI 层的引入，简化了操作，提高了开发人员开发调试程序的效率。

## 本地、远程调试

### 使用`JDB`进行调试

#### 本地调试

在使用`JDB`进行远程调试之前，我们应该先试验一下，使用`JDB`进行本地调试。

首先编写一个测试类`TestJdb.java`

```java
public class TestJdb {
    public static void main(String[] args) {
        int a = 1;
        int b = 2;
        int c = a + b;
        int d = c + b;
        int e = a + d;
        System.out.println("a=" + a + ",b=" + b + ",c=" + c + ",d=" + d + ",e=" + e);
    }
}
```
使用javac编译得到`TestJdb.class`,使用命令`jdb TestJdb`即可进入调试模式,使用`stop`命令可设置断点,使用`run`命令即可运行程序

```
>jdb TestJdb
正在初始化jdb...
> stop at TestJdb:5
正在延迟断点TestJdb:5。
将在加载类后设置。
> run
运行TestJdb
设置未捕获的java.lang.Throwable
设置延迟的未捕获的java.lang.Throwable
>
VM 已启动: 设置延迟的断点TestJdb:5

断点命中: "线程=main", TestJdb.main(), 行=5 bci=4
5            int c = a + b;
```

**jdb命令：**

| 名称      | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| help or ? | 最重要的**JDB**命令; 它会显示一个包含简要说明的已识别命令列表。 |
| run       | 启动**JDB**并设置必要的断点后，可以使用此命令开始执行并调试应用程序。 |
| cont      | 在断点，异常或步骤之后继续执行已调试的应用程序。             |
| print     | 显示Java对象和原始值。                                       |
| dump      | 对于原始值，此命令与print相同。 对于对象，它会打印对象中定义的每个字段的当前值。 包括静态和实例字段。 |
| threads   | 列出当前正在运行的线程。                                     |
| thread    | 选择一个线程作为当前线程。                                   |
| where     | 转储当前线程的堆栈。                                         |

**如何设置断点**

```
stop at <class>:<line_number> 或
     stop in <class>.<method_name>[(argument_type,...)]
```

**如何打印参数**

通过 `print` 或 `dump` 命令查看此时指定变量的值，`print` 用于查看简单类型，`dump` 用于查看对象类型

```
main[1] print a
 a = 1
main[1] print this
com.sun.tools.example.debug.expr.ParseException: No 'this'.  In native or static method
 this = 空值
main[1] dump this
com.sun.tools.example.debug.expr.ParseException: No 'this'.  In native or static method
 this = 空值
main[1]
```

**如何执行方法**

通过 `step`、`next` 或 `cont` 命令继续执行程序：

- `step` 命令相当于 eclipse 当中的 F5，如果当前语句是另一个方法调用时，会进入那个方法当中
- `next` 命令相当于 F6，只会逐行执行，不会进入被调用的其它方法
- `cont` 命令相当于 F8，从当前行一直执行到下一个断点，如果没有就一直执行到程序结束

#### 远程调试

使用`jdb`的`attach`命令，可以链接到已经处于运行状态的目标JVM。具体命令为

```
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=<PORT> <主类的全路径名>
```

其中 `address` 参数可选，如果不指定的话会随机分配一个可用端口。

如果是使用`tomcat`部署的`war`包工程，则需要在tomcat的启动脚本（`catalina.sh`）里添加下面的参数：

```
JAVA_OPTS="$JAVA_OPTS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005"
```

同时需要关闭服务器防火墙，或者开发调试端口。

**一个连接示例：**

首先使用下面的命令启动服务：

```sh
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n -jar helloworld-1.0.0-SNAPSHOT.jar
```

启动成功后，日志中会打印出提供给 jdb 连接的端口，因为我们之前未指定，所以这里随机分配了一个

```sh
dereck-mbp:target Dereck$ java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n -jar helloworld-1.0.0-SNAPSHOT.jar 
Listening for transport dt_socket at address: 51750
```

通过 `jdb` 命令连接到正在运行的 JVM

```sh
dereck-mbp:~ Dereck$ jdb -connect com.sun.jdi.SocketAttach:port=51750
设置未捕获的java.lang.Throwable
设置延迟的未捕获的java.lang.Throwable
正在初始化jdb...
> 
```

后续调试方式与`本地调试`方法一致。


### 使用开发工具进行调试

常见的开发工具有`eclipse`、`myeclipse`、`idea`，对于开发工具的本地调试，因为可视化话操作比较简单，顾本文不再赘述。这里主要说明一下如果使用可视化工具进行远程调试，下面以`idea`为例。

在进行调试之前，同意需要在启动服务的时候添加调试相关的参数，详细见`jdb`远程调试部分，示例为：

```
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```

打开idea中的run/debug configurations, 选择remote类型，地址配置为服务器地址，端口配置为上述配置参数中的address，配置如图：

![idea调试截图](https://static.jiangliuhong.top/blogimg/tools/idea%E8%B0%83%E8%AF%95%E6%88%AA%E5%9B%BE.png)