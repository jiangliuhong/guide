---
title: JavaScript基础学习
date: 2019-05-23 09:49:02
categories: 随笔
tags: 
 - JavaScript
---

# JavaScript基础学习

> JavaScript，简称JS，是一种高级的、解释执行的编程语言。JavaScript是一门基于原型、函数先行的语言，是一门多范式语言，它支持面向对象编程，以及函数式编程。它被世界上的绝大多数网站使用，也被世界主流浏览器支持。

## 作用域

在JavaScript中，对象和函数同样也是变量。

在JavaScript中，作用域可访问变量，对象，函数的集合。

对于变量而言，在JavaScript中有两个范围：全局和局部。在函数定义之外的声明的变量属全局变量，它在整个应用程序（也就是整个页面的js）都可以访问；反之，函数定义之内的声明的变量属局部变量，每当函数执行时，会创建变量，当函数执行完成后，都会销毁变量，并且该函数之外的内容无法访问该变量。

在JavaScript中同样也支持块作用域。

块作用域主要有for,if,with,tyr/catch,let,const。

```javascript
for(var i=o;i<10;i++){
    console.log(i);
}
```

因为在for循环内部定义了i变量，顾i变量只能在for循环体内使用。

```javascript
var foo = true;
if (foo) {
 var bar = foo * 2;
 bar = something( bar ); 
 console.log( bar );
}
```

bar变量仅声明在if的上下文中，因此在使用bar变量的时候，只能在if的方法体内使用。

with语句的作用是将代码的作用域设置到一个特定的作用域中，with 通常被当作重复引用同一个对象中的多个属性的快捷方式，可以不需要重复引用对象本身。

比如：

```javascript
var obj = {
 a: 1,
 b: 2,
 c: 3
};
// 单调乏味的重复 "obj"
obj.a = 2;
obj.b = 3;
obj.c = 4;
// 简单的快捷方式
with (obj) {
 a = 3;
 b = 4;
 c = 5;
}
```

try/catch相对比较容易理解，就是try与catch创建的{}代码块属于一个块作用域。

let是ES6提出的一个新的声明变量的关键字，let关键字可以将变量绑定在其所在的任意作用域中，通常是{}中，但这是一种隐式行为。

const，与let相似的，const也是ES6引入的一个新的声明变量的关键字，但其值是固定的，即常量。

## 基础类型

在JavaScript中共有6中基本数据类型：

- Undefined：表示一个对象未定义
- Null：对象已经定义，但这个对象是空的，即空指针对象
- Boolean：布尔类型，该类型只有两个值true、false
- Number：数字类型，包括整数、浮点数，当然也支持十进制、八进制、十六进制表示。另外Number还有一个特殊的值，NaN，该值用于表示一个本来要返回数值的对象，但未返回数值的情况
- String：字符串类型
- Symbol：ES6新引入的数据类型，它表示一个独一无二的值，其作用是放置属性名冲突

基础类型比较：

在JavaScript中，比较两个基础类型，JS会自动读数据进行隐式转换，比如:

```javascript
var a = 1;
var b = true;
console.log(a == b);    // true
console.log(a === b);   // false
```

在JavaScript中， ==，双等号只进行值比较，如果两者数据不同意，则会自动转成同意的数据格式进行比较，当然如果转换格式失败，则会抛出异常。===，三等又称强等号，三等不仅会验证值是否一致，同时也是验证数据格式是否一致。

基础类型的变量存在在栈内存(Stack)中。

## 引用类型

JavaScript中的引用类型，即Object类型，对象类型，对象可以是一个类型的实例化对象，也可以是一组数据源，也可以是一个功能函数。在JavaScript中，Object又有许多子类型，如：Array，Date，RegExp，Function等。

RegExp类型，即正则表达式，在JavaScript中，它是用于描述字符模式的对象，正则表达式用于对字符模式匹配、检索替换，是操作字符串执行模式匹配的强大工具。其使用方式如下：

```javascript
//验证字符串是否全数字
var re = new RegExp("^[0-9]*$");
var str = '123213123'
re.test(str)//true
```

对于RegExp支持的字符串方法有：search、match、replace、split。

Function，函数类型，每个函数都是Function类型的实例。

对于函数，其返回值比较特殊，如果其方法体内没有返回值，则返回一个undefined。

```javascript
function test1(){		
}
function test2(){
    return 1;
}
function test3(){
    return ;
}
var t1 = test1();
var t2 = test2();
var t3 = test3();
console.log(t1);//undefined
console.log(t2);//1
console.log(t3);//undefined
```

对于引用类型的值是按照引用访问的。所以在比较引用类型的时候，双等号与三等号作用相同，比较的是两个对象的引用地址，引用地址相同，才会返回true。例如：

```javascript
var obj1 = {};    // 新建一个空对象 obj1
var obj2 = {};    // 新建一个空对象 obj2
console.log(obj1 == obj2);    // false
console.log(obj1 === obj2);   // false
```

引用类型的值存在堆内存(Heap)中。

虽然引用类型保存在堆内存中，但是JavaScript不能直接操作堆内存，所以在JavaScript中，栈内存中保存了变量标识符和指向堆内存中该对象的指针，堆内存中保存了对象的内容。

## 检测类型

在JavaScript中，有两个较特殊的关键字，分别为typeof、instanceof。

typeof主要用来检测一个变量是否为基本的数据类型。

```javascript
var a;
typeof a;    // undefined
a = null;
typeof a;    // object
a = true;
typeof a;    // boolean
a = 666;
typeof a;    // number 
a = "hello";
typeof a;    // string
a = Symbol();
typeof a;    // symbol
a = function(){}
typeof a;    // function
a = [];
typeof a;    // object
a = {};
typeof a;    // object
a = /aaa/g;
typeof a;    // object   
```

instanceof主要用来检测构造函数的prototype属性所指向的对象是否存在于另一个检测对象的原型链（关于原型链见下文）上。

```javascript
({}) instanceof Object              // true
([]) instanceof Array               // true
(/aa/g) instanceof RegExp           // true
(function(){}) instanceof Function  // true
```

