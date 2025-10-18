---
title: Java线程安全
categories: 
    - Java
date: 2019-08-26 20:44:39
tags:
    - Java多线程
---

# 线程安全

> 一个方法或者一个实例可以在多线程环境中使用而不会出现问题

## 线程安全实现方式

### 悲观锁

#### synchronized

使用synchronized是通过互斥的方式保证同步的，它对于同一条线程来说是可重入的；其次它是阻塞的。synchronized会将线程阻塞，当获得锁时会唤醒线程，将线程从用户状态转为内核状态，该操作会消耗大量的资源，顾synchronized是一个重量级锁。

#### ReentrantLock

ReentrantLock是一个可重入锁。它与synchronized的区别是，它需要显示的去调动`lock`与`unlock`方法，手动去加锁并释放锁，一般释放锁的方法写在`finally`语句块中。

### 乐观锁

#### CAS（Compare And Swap）

>CAS是原子操作，保证并发安全，不能保证并发同步
>
>CAS是CPU的一个指令
>
>CAS是非阻塞的、轻量级的乐观锁

原理：CAS比较并替换，就是将内存值更新为需要的值，但是有个条件，内存值必须与期望值相同。

最佳应用：`java.util.concurrent.atomic`包下的原子操作类

- 原子更新基本类型
- 原子更新数组
- 原子更新引用
- 原子更新字段

CAS优点：

- 乐观锁，通过CPU指令实现，性能高

CAS缺点：

- 自旋时间长，消耗CPU资源
- 非公平锁

### synchronized使用方法

> synchronized修饰的对象主要为四种：
>
> - 修饰代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，作用的对象是调用这个代码块的对象；
> - 修饰方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象； 
> - 修饰一个静态方法，其作用的范围是整个静态方法，作用的对象是这个类的所有对象； 
> - 修饰一个类，其作用的范围是synchronized后面括号括起来的部分，作用主的对象是这个类的所有对象。

#### 修饰代码块

被修饰的代码块被称为同步语句块，起作用范围是{}中的代码，作用的对象是这个代码块的对象。

```java
public class UserService {
    
    public void addUser(User user){
        synchronized(this){
           System.out.println("add user");
        }
    }
}
```

多个线程访问同一个对象的代码块时，线程被阻塞；

多个线程访问不同对象的代码块，线程不会被阻塞。

即synchronized修饰的是一个对象，每个访问这个对象的线程都将被阻塞，直到该对象的锁被释放。

如果没有明确的对象加锁，可以在类中定义一个常量来实现：

```java
public class UserService {
    
    private String lock = "lock";
    
    public void addUser(User user){
        synchronized(lock){
           System.out.println("add user");
        }
    }
}
```

####  修饰方法

```java
public class UserService {
    public synchronized void updateUser(User user){
         System.out.println("update user");
    }
}
```

synchronized修饰方法时，它锁定的是调用这个同步方法的对象。即一个对象在不同的线程中调用该方法将被阻塞。

#### synchronized修饰一个静态的方法

```java
public class UserService {
    public synchronized static void updateUser(User user){
         System.out.println("update user");
    }
}
```

synchronized作用的对象是一个静态方法，则它去得是类的所，该类素有的对象同一把锁，所有对象都会被阻塞。

#### synchronized作用于一个类

```java
public class UserService {
    public void updateUser(User user){
        synchronized(UserService.class){
        	 System.out.println("update user");
        }
    }
}
```

synchronized作用于一个类时，是给这个类加锁，这个类的所有对象用的是同一把锁。

#### 小结

- 无论synchronized关键字加在方法上还是对象上，如果它作用的对象是非静态的，则它取得的锁是对象；如果synchronized作用的对象是一个静态方法或一个类，则它取得的锁是对类，该类所有的对象同一把锁。
- 每个对象只有一个锁（lock）与之相关联，谁拿到这个锁谁就可以运行它所控制的那段代码。
- 实现同步是要很大的系统开销作为代价的，甚至可能造成死锁，所以尽量避免无谓的同步控制。

## 线程安全的集合

### 最原始的集合

> 原始的线程安全集合都是通过在方法上添加`synchronized`关键字，该方法效率较低。

**Vector：**长度可变的数组

**HashTable：**线程安全的HashMap

### concurrent包中的集合

**ConcurrentHashMap：**线程安全的HashMap，jdk1.8之前采用Segment分段锁，jdk1.8取消了分段锁，直接在table元素上加锁，实现对每一行进行加锁。

**CopyOnWriteArrayList：**具有写锁

**CopyOnWritArraySet：**具有写锁

**ConcurrentSkipListMap：**利用跳表实现的有序的、支持高并发的Map

**ConcurrentSkipListSet：**利用跳表实现的有序的、支持高并发的Set

**ConcurrentLinkedQueue：**高并发场景下的队列

**ConcurrentLinkedDeque：**高并发场景下的双端队列

## 线程安全Queue

在Java多线程应用中，Queue主要分为两种：

- BlockingQueue：阻塞队列
- ConcurrentLinkedQueue：高性能队列

#### BlockingQueue

`BlockingQueue`是一个接口，定义了一套阻塞队列的规则。它的常用方法为：

- add(e) remove() element() 方法不会阻塞线程。当不满足约束条件时，会抛出IllegalStateException 异常。例如：当队列被元素填满后，再调用add(e)，则会抛出异常。
- offer(e) poll() peek() 方法即不会阻塞线程，也不会抛出异常。例如：当队列被元素填满后，再调用offer(e)，则不会插入元素，函数返回false。
- 要想要实现阻塞功能，需要调用put(e) take() 方法。当不满足约束条件时，会阻塞线程。

##### ArrayBlockingQueue

`ArrayBlockingQueue`是基于数组的阻塞队列实现，在ArrayBlockingQueue内部，维护了一个定长数组，以便缓存队列中的数据对象，其内部没实现读写分离，也就意味着生产和消费不能完全并行，长度是需要定义的，可以指定先进先出或者先进后出，也叫有界队列，在很多场合非常适合使用。

##### LinkedBlockingQueue

`LinkedBlockingQueue`是基于链表的阻塞队列，同ArrayBlockingQueue类似，其内部也维持着一个数据缓冲队列〈该队列由一个链表构成），LinkedBlockingQueue之所以能够高效的处理并发数据，是因为其内部实现采用分离锁（读写分离两个锁），从而实现生产者和消费者操作的完全并行运行,他是一个无界队列。

##### SynchronousQueue

`SynchronousQueue`是一种没有缓冲的队列，生产者产生的数据直接会被消费者获取并消费。

##### PriorityBlockingQueue

`PriorityBlockingQueue`是基于优先级的阻塞队列（优先级的判断通过构造函数传入的Compator对象来决定，也就是说传入队列的对象必须实现Comparable接口），在实现PriorityBlockingQueue时，内部控制线程同步的锁采用的是公平锁，他也是一个无界的队列。

##### DelayQueue

`DelayQueue`：带有延迟时间的`Queue`，其中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。`DelayQueue`中的元素必须实现`Delayed`接口，`DelayQueue`是一个没有大小限制的队列，应用场景很多，比如对缓存超时的数据进行移除、任务超时处理、空闲连接的关闭等等。

##### ConcurrentLinkedQueue

`ConcurrentLinkedQueue`是一个适用于高并发场景下的队列，通过无锁的方式，实现了高并发状态下的高性能，通常ConcurrentLinkedQueue性能好于BlockingQueueo它是一个基于链接节点的无界线程安全队列。该队列的元素遵循先进先出的原则。头是最先加入的，尾是最近加入的，该队列不允许null元素。

其核心方法为：

- `add`、`offer`：将元素加入队列，两个方法没有区别
- `poll`：取头元素节点，该方法会删除元素
- `Peek`：取头元素节点

## java.utils.cuncurrent下的锁

### ReentrantLock 

- 一个可重入的互斥锁 Lock
- ReentrantLock 将由最近成功获得锁，并且还没有释放该锁的线程所拥有
- 此类的构造方法接受一个可选的公平参数。当设置为`true`时，在多个线程的争用下，这些锁倾向于将访问权授予等待时间最长的线程。采用默认设置（使用不公平锁）。
- 使用公平锁的程序在许多线程访问时表现为很低的总体吞吐量（即速度很慢，常常极其慢），优点是在获得锁和保证锁分配的均衡性时差异较小。

### ReadWriteLock

`ReadWriteLock `维护了一对相关的锁，一个用于只读操作，另一个用于写入操作。只要没有 writer，读取锁可以由多个 reader 线程同时保持。写入锁是独占的。

- 与互斥锁相比，读-写锁允许对共享数据进行更高级别的并发访问。
- 一次只有一个线程（*writer* 线程）可以修改共享数据；在许多情况下，任何数量的线程可以同时读取共享数据（*reader* 线程）
- 与互斥锁相比，使用读-写锁所允许的`并发性`增强将带来更大的性能提高

### **ReentrantReadWriteLock** 

`ReentrantReadWriteLock`是`ReadWriteLock `的实现类。它具有以下属性：

- 获取顺序：`ReentrantReadWriteLock`不会将读取者优先或写入者优先强加给锁访问的排序
- 公平性：默认为非公平锁，可以在构造函数中设置为公平锁
- 重复性：可重入。
- 锁降级：重入还允许从写入锁降为读取锁。实现方式是先获取写入所，然后获取读取锁，最后释放写入锁。即写入线程可以获取读取锁，读取锁中不能获取写入锁。

### Stampedlock

> StampedLock是并发包里面jdk8版本新增的一个锁，该锁提供了三种模式的读写控制，三种模式分别如下

**写锁writeLock：**排它锁（独占锁），同时只有一个线程可以获取该锁，当一个线程获取该锁后，其它请求的线程必须等待，当目前没有线程持有读锁或者写锁的时候才可以获取到该锁，请求该锁成功后会返回一个stamp票据变量用来表示该锁的版本，当释放该锁时候需要unlockWrite并传递参数stamp。

**悲观读锁readLock：**共享锁，在没有线程获取独占写锁的情况下，同时多个线程可以获取该锁，如果已经有线程持有写锁，其他线程请求获取该读锁会被阻塞。

**乐观读锁tryOptimisticRead：**乐观锁，运用CAS原理。适用于读多写少的场景，因为获取读锁只是使用与或操作进行检验，不涉及CAS操作，所以效率会高很多。