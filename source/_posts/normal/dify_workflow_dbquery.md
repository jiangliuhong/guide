---
title: 使用dify构建一个查询数据库的工作流
date: 2025-10-29 21:50:45
categories:
  - 随笔
tags: AI
cover: https://assets-docs.dify.ai/2025/05/d05cfc6ebe48f725d171dc71c64a5d16.svg
---

## 前提条件

### dify配置

dify 1.9.2，安装教程见[官网](https://docs.dify.ai/zh-hans/getting-started/install-self-hosted/docker-compose)

模型供应商添加一个深度求索（当然也可以使用其他模型）

安装插件“数据库查询”，插件地址：[跳转地址](https://marketplace.dify.ai/plugins/junjiem/db_query?source=http%253A%252F%252F121.37.246.252%253A81&amp;theme=system)

![image](https://static.jiangliuhong.top/images/picgo/20251029212230.png)

### 数据库

准备一个数据库结构描述文件，这里我准备了一个excel,然后又两个工作表，分别如下:

**表信息：**

| 模式 | 表            | 备注         |
| ------ | --------------- | -------------- |
| wos  | USERS         | 用户信息     |
| wos  | USER_REGISTER | 用户注册信息 |

**字段信息：**

| 模式 | 表            | 字段          | 类型         | 备注     | 关联外键     |
| ------ | --------------- | --------------- | -------------- | ---------- | -------------- |
| wos  | USERS         | ID            | VARCHAR(32)  | 主键     |              |
| wos  | USERS         | USERNAME      | VARCHAR(100) | 用户名   |              |
| wos  | USERS         | PASSWORD      | VARCHAR(100) | 密码     |              |
| wos  | USER_REGISTER | ID            | VARCHAR(32)  | 主键     |              |
| wos  | USER_REGISTER | USER_ID       | VARCHAR(100) | 用户ID   | wos.USERS.ID |
| wos  | USER_REGISTER | REGISTER_DATE | VARCHAR(100) | 注册时间 |              |

插入数据：

```sql
INSERT INTO wos.USERS (ID, USERNMAE, PASSWORD) VALUES ('1', 'user1', 'password1');
INSERT INTO wos.USERS (ID, USERNMAE, PASSWORD) VALUES ('2', 'user2', 'password2');
INSERT INTO wos.USER_REGISTER (ID, USER_ID, REGISTER_DATE) VALUES ('1', '1', '2025-10-10');
INSERT INTO wos.USER_REGISTER (ID, USER_ID, REGISTER_DATE) VALUES ('2', '2', '2025-10-20');
```

## 编写流程

![](https://static.jiangliuhong.top/images/picgo/20251029204503.png "完整流程")

### 开始节点

添加两个输入变量

- input，类型为文本
- ddl，类型为文件

![](https://static.jiangliuhong.top/images/picgo/20251029211541.png)

这里我为了避免运行的时候上传文件，所以对`ddl`设置了默认值，并且将其隐藏了。实际应用中可以在执行工作流的时候传递。

### 提取关键字

上下文选择`开始节点`的`input`变量，输入下面的提示词

```
你是一个关键词提取器，根据用户输入的{{#1761707244261.input#}}，解析出用户想要查询的数据库表名和查询条件，如果可能涉及多个表，你需要全部返回，直接返回关键词即可
```

![](https://static.jiangliuhong.top/images/picgo/20251029211743.png)

### 提取表结构

上下文选择`开始节点`的`ddl`变量

![](https://static.jiangliuhong.top/images/picgo/20251029211942.png)

### 构建SQL

上下文选择`提取表结构`的`ddl`变量，输入下面的提示词

```
{{#context#}}是对MySQL一个表或多个表的描述（多个表使用外键进行关联），根据这个描述编写一个对这个表的查询SQL，直接返回纯文本的SQL，不要用markdown格式，例如 SELECT column from table。请基于{{#1761720485233.text#}}表结构，以及用户查询的表和条件{{#1761707260422.text#}}组装一个查询sql
```

![](https://static.jiangliuhong.top/images/picgo/20251029212424.png)

### SQL查询

在该节点需要录入你的数据库信息，该插件目前支持的数据库有： MySQL,Oracle,SQLServer,PostgreSQL

SQL查询语句选择`构建SQL`节点的`text`变量，输出格式这里选择`JSON`

![](https://static.jiangliuhong.top/images/picgo/20251029212641.png)

### 结束节点

输出变量选择`SQL查询`节点的输出信息

![](https://static.jiangliuhong.top/images/picgo/20251029213101.png)

## 执行测试

输入“查询注册2025年10月15日之后注册的用户”进行测试，返回结果显示查询到了ID为2的用户记录，查询结果正确。	

![](https://static.jiangliuhong.top/images/picgo/20251029214626.png)

通过追踪查看`构建SQL`节点的输出结果如下：

```
{
  "text": "SELECT u.ID, u.USERNMAE, ur.REGISTER_DATE \nFROM wos.USERS u \nJOIN wos.USER_REGISTER ur ON u.ID = ur.USER_ID \nWHERE ur.REGISTER_DATE > '2025-10-15'",
}
```