---
title: npm源设置
date: 2020-04-08 22:16:45
categories:
  - 随笔
tags: 
  - nodejs
cover: https://static.jiangliuhong.top/blogimg/other/nodejs-npm.jpg
---

## 将npm的源替换成淘宝的源

```
npm config set registry http://registry.npm.taobao.org/
```
**修改后面的地址，也可以更换为私有源**

## 修改源地址为官方源

```
npm config set registry https://registry.npmjs.org/
```

## 安装cnpm

```
npm install -g cnpm --registry=https://registry.npm.taobao.org
```