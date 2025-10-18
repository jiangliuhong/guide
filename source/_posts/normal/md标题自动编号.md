---
title: hexo中markdown文档自动编号
date: 2020-03-26 00:17:00:00
categories: 随笔
---

# hexo中markdown文档自动编号

## css实现原理

有时候我们需要对文章进行编号，例如：

```
1 标题1
1.1 标题1.1
1.1.1 标题1.1.1
1.2 标题1.2
```

但是这在`hexo`是不支持的，需要自己开发，查阅了资料，发现可以利用css的计数器(counter)来实现此功能。


在css里，我们可以声明一个计数器，假设其名称为counter_xxx，那么我们可以使用counter()函数获得它当前的值。

```
counter(counter_xxx);
```

与此同时，我们可以使用counter-reset属性指定需要重置的计数器，使用counter-increment属性指定计数加一的计数器。

```
h1 {
	counter-increment: counter_h1;
	counter-reset: counter_h2;
}
```

如上代码所示，每有一个h1出现时，计数器counter_h1的值就会加一，而counter_h2的值就会重置为0。我们刚好可以利用这个，配合伪类选择器:before来实现标题的自动编号。

## 具体实现

**代码详情：**

```css
h2 {
	counter-increment: counter_h2;
	counter-reset: counter_h3;
}
h2 .headerlink:before {
	content: counter(counter_h2)"　";
}

h3 {
	counter-increment: counter_h3;
	counter-reset: counter_h4;
}

h3 .headerlink:before {
	content: counter(counter_h2)"."counter(counter_h3)"　";
}

h4 {
	counter-increment: counter_h4;
	counter-reset: counter_h5;
}

h4 .headerlink:before {
	content: counter(counter_h2)"."counter(counter_h3)"."counter(counter_h4)"　";
}
```

**注意：**

- `h4 .headerlink:before`这个是具体展现的`css`，你可根据自生情况来说。

- 这里我是从`h2`开始编号的，如果需要，你也可以从`h1`开始编号。

**展现效果：**

## h2标题

### h3标题

#### h4标题

#### h4标题

#### h4标题

### h3标题

#### h4标题

#### h4标题

## h2标题

### h3标题

#### h4标题



