---
title: java8新特性
categories: 
    - Java
date: 2019-08-25 20:44:39
tags:
    - tools
cover: https://static.jiangliuhong.top/blogimg/java/java8-logo.jpeg
---

# Java8新特性

## 接口默认方法与静态方法

### 接口默认方法

实现方式：

```java
public interface UserInterface {
    default void addUser(){
        System.out.println("add user");
    }
}
```

在接口定义中使用`default`关键字即可定义一个自带方法内容的接口方法，子类可以不实现该方法。

#### 接口冲突

接口是允许实现多个接口的，试想如果有两个接口，分别为`UserInterface1`、`UserInterface2`，他们均定义了一名名为`addUser`的默认方法，此时，如果有一个子类同时实现了这两个接口，实现方调用`addUser`则产生冲突。

#### 优点

可以在不破坏代码的前提下扩展原有库的功能。它通过一个很优雅的方式使得接口变得更智能，同时还避免了代码冗余，并且扩展类库。

#### 缺点

使得口作为协议，类作为具体实现的界限开始变得有点模糊。如果继承关系较多，可能增加一些开发成本。

### 静态方法

```java
public interface UserInterface {

    static String getUser(){
        System.out.println("this user");
        return "user";
    }

}
```

## Lambda 表达式

`Lambda`表达式（也称为闭包）是整个Java 8发行版中最受期待的在Java语言层面上的改变，Lambda允许把函数作为一个方法的参数（即：**行为参数化**，函数作为参数传递进方法中）。

一个`Lambda`可以由用逗号分隔的参数列表、`–>`符号与函数体三部分表示。

遍历List的两种方式：

```java
@Test
public void t1(){
  List<String> list = new ArrayList<>();
  list.add("A");
  list.add("B");
  list.add("C");
  System.out.println("普通遍历：");
  for(String str : list){
    System.out.println(str);
  }
  System.out.println("java8遍历：");
  list.forEach(str->{
    System.out.println(str);
  });
}
```

## 函数式接口

函数式接口是指一个抽象方法的接口，每一个该类型的Lambda表达式都会被匹配到这个抽象方法。

一个`Lambda`表达式就是一个抽象方法的实现。

`@FunctionalInterface`定义的接口都可以使用在`Lambda`表达式上。

### Java8自带的函数式接口

**Comparator (比较器接口)**

```java
Comparator<Person> comparator = (p1, p2) -> p1.firstName.compareTo(p2.firstName);
```

**Consumer (消费型接口)**

`Consumer`接口表示执行在单个参数上的操作。 

扩展：

- BiConsumer
- DoubleConsumer
- IntConsumer
- LongConsumer
- ObjDoubleConsumer
- ObjIntConsumer
- ObjLongConsumer

**Supplier（供应型接口）**

`Supplier`接口是不需要参数并返回一个任意范型的值

扩展：

- BooleanSupplier
- DoubleSupplier
- IntSupplier
- LongSupplier

**Predicate（断言型接口）**

`Predicate`接口只有一个参数，返回`boolean`类型。

扩展：

- BiPredicate
- DoublePredicate
- IntPredicate
- LongPredicate

**Function (功能型接口)**

`Function`接口有一个参数并且返回一个结果，并附带了一些可以和其他函数组合的默认方法（`compose`, `andThen`）

扩展：

- BiFunction
- DoubleFunction
- IntFunction
- LongFunction
- ToDoubleFunction
- ToDoubleBiFunction

**Operator**

`Operator`其实就是`Function`，函数有时候也叫作算子。算子在Java8中接口描述更像是函数的补充，和上面的很多类型映射型函数类似。

- UnaryOperator
- BinaryOperator

**示例代码**

```java
public class User {

    private Work work;

    public Work getWork() {
        return work;
    }

    public void setWork(Work work) {
        this.work = work;
    }

    public void eat(String food, Consumer<String> consumer) {
        consumer.accept(food);
    }

    public void eat(String food, Supplier<String> supplier) {
        String s = supplier.get();
        System.out.println("Supplier ->> " + food + ":" + s);
    }

    public void eat(String food, Predicate<String> predicate) {
        if (predicate.test("admin")) {
            System.out.println("Predicate ->> admin eat " + food);
        }
    }

    public void work(Integer month,Function<Work, Integer> function) {
        Integer amount1 = function.compose(w-> {
            Work ww = new Work();
            ww.setSalary(((Work)w).getSalary()*month);
            return ww;
        }).apply(this.getWork());
        System.out.println("Function compose ->> amount:"+ amount1);
        Integer amount2 = function.andThen(salary -> salary * month).apply(this.getWork());
        System.out.println("Function andThen ->> amount:"+ amount2);
    }

    public static void main(String[] args) {
        User user = new User();
        user.eat("banana", (Consumer<String>)s -> System.out.println("Consumer ->> food is : " + s));
        user.eat("banana", (Predicate<String>)s -> s.equals("admin"));
        user.eat("apple", () -> "sdf");
        Work work = new Work();
        work.setSalary(100);
        user.setWork(work);
        user.work(8, s -> {
            Integer salary = s.getSalary();
            //奖金
            Integer bons = 100000;
            return salary + bons;
        });
        UnaryOperator<Integer> increment = x -> x + 1;
        System.out.println("UnaryOperator ->> " + increment.apply(2));
        BinaryOperator<Integer> add = (x, y) -> x + y;
        System.out.println("BinaryOperator ->> " + add.apply(2, 3));

        BinaryOperator<Integer> min = BinaryOperator.minBy((o1, o2) -> o1 - o2);
        System.out.println("BinaryOperator ->> " + min.apply(2, 3));
    }
}

public class Work {

    private Integer salary;

    public Integer getSalary() {
        return salary;
    }

    public void setSalary(Integer salary) {
        this.salary = salary;
    }
}
```

**示例输出**

```text
Consumer ->> food is : banana
Predicate ->> admin eat banana
Supplier ->> apple:sdf
Function compose ->> amount:100800
Function andThen ->> amount:800800
UnaryOperator ->> 3
BinaryOperator ->> 5
BinaryOperator ->> 2
```

###  自定义函数式接口

自定义函数式接口只需要在接口中添加`@FunctionalInterface`注解，并为这个接口提供至少一个公共方法即可。下面示例代码将自定义个将map转为list的函数式接口：

```java
import java.util.List;
import java.util.Map;

@FunctionalInterface
public interface ListToMapFunction<K,V> {

    public Map<K,V> apply(List<V> list);

}
```

## 方法引用

首先列举一下java8中使用stream将list转为map的代码：

```java
public static void main(String[] args) {
        List<User> users = new ArrayList<>();
        users.add(new User(1));
        users.add(new User(2));
        users.add(new User(3));
        users.add(new User(4));
        users.add(new User(5));
        users.stream().collect(Collectors.toMap(u->u.getId(),u->u));
    }
```

在java8中我们可以直接通过方法应用来简写`Lambda`表达式中已经存在的方法，修改为：

```java
users.stream().collect(Collectors.toMap(User::getId,u->u));
```

其中`User::getId`就是方法引用，方法引用的操作符是双冒号`::`。

**方法引用**是用来直接访问类或者实例的已经存在的方法或者构造方法。方法引用提供了一种引用而不执行方法的方式，它需要由兼容的函数式接口构成的目标类型上下文。计算时，方法引用会创建函数式接口的一个实例。当Lambda表达式中只是执行一个方法调用时，不用Lambda表达式，直接通过方法引用的形式可读性更高一些。方法引用是一种更简洁易懂的Lambda表达式。

方法引用有四种写法：

- 引用静态方法: ContainingClass::staticMethodName
- 引用某个对象的实例方法: containingObject::instanceMethodName
- 引用某个类型的任意对象的实例方法:ContainingType::methodName
- 引用构造方法: ClassName::new

## Stream流

`Stream`是java8新增的类，主要是用来补充集合类。它代表数据流，流中的数据元素的数量可能是有限的，也可能是无限的。

`Stream`与集合类的区别：集合类关注有限数量的数据访问和有效管理，而`Stream`是在数据源上执行可计算的操作。在一个流中，可以执行一个或者多个中间操作，再执行一个最终操作来返回结果。

中间操作有：

- `filter`
- `map`：归类为一组数据
- `flatMap`：将map生成的流合并成单个流
- `peek`:与`map`类型，区别是它接受一个没有返回值的表达式
- `distinct`:去重
- `sorted`:对流中的元素进行排序
- `limit`:减少流的大小
- `substream`

终止操作有：

- `forEach`：遍历该流中的每个元素
- `toArray`
- `reduce`:用于对两个顺序流的计算
- `collect`:方法是终端操作，这是通常出现在管道传输操作结束标记流的结束
- `min`
- `max`
- `count`
- `anyMatch`:是否存在任意一个元素满足条件（返回布尔值）
- `allMatch`:是否所有元素都满足条件（返回布尔值）
- `noneMatch`:是否所有条件都不满足（返回布尔值）
- `findFirst`:查找到第一个就返回Optional
- `findAny`:查找到任意一个就返回Optional
- `java.util.stream.Collectors`

## Optional

`Optional`实际上是个容器：它可以保存类型T的值，或者仅仅保存null。`Optional`提供很多有用的方法，这样我们就不用显式进行空值检测。

方法说明：

- `isPresent`:为空返回true,否返回false
- `orElse`:为空，则返回默认值
- `orElseGet`:为空，调动get回调函数