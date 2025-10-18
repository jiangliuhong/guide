---
title: 学习制作nacos的rpm安装包
date: 2021-01-29 15:16:45
categories:
  - 随笔
tags: 
  - nacos
cover: https://static.jiangliuhong.top/blogimg/other/nacos_title.png 
---

# 学习制作nacos的rpm安装包

> 公司的项目需要部署在国产化服务器上，该国产化服务器只能通过`rpm`安装软件，可nacos官方并未提供，google搜索无果，于是只有自己学习制作了。
>
> 参考地址：https://juejin.cn/post/6844903895873880077#heading-11

## 安装fpm
FPM是一个用Ruby编程语言构建的工具，所以在安装FPM之前需要安装ruby
### 安装环境
```
$ yum install rubygems ruby-devel rubugems-devel gcc rpm-build make -y
```
### 更换为国内的gem源
```
$ gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
$ gem sources -l
https://gems.ruby-china.com

#更新gem版本
$ gem update --system
```
### 使用gem安装FPM

```
$ gem install --no-ri --no-rdoc fpm
```

在安装过程中，可能会提示如下错误：

```
ERROR:  While executing gem ... (Gem::DependencyError)
    Unable to resolve dependencies: fpm requires ffi (~> 1.12.0); ruby-xz requires ffi (~> 1.9); childprocess requires ffi (>= 1.0.11, ~> 1.0)
```

一般是由于ruby版本过低导致。查看ruby版本：

```
ruby -v
ruby 2.0.0p648 (2015-12-16) [x86_64-linux]
```

升级版本

```
# 安装rvm
curl -L get.rvm.io | bash -s stable
# 加载环境变量
source /etc/profile.d/rvm.sh
# 查看版本
rvm -v
```

在执行`curl`命令的时候，可能会报这样的错误：

```
curl: (7) Failed connect to raw.githubusercontent.com:443; Connection refused
```

针对这一错误，只需要修改host即可，或者是增加代理

```
sudo vim /etc/hosts
```

增加以下内容：

```
199.232.68.133 raw.githubusercontent.com
```

## 打包nacos

打包命令：

```
fpm -s dir  \
-t rpm \
-n nacos \
-v 1.4.1 \
-C /home/nacos \
-p /home/rpm \
--description 'Nacos rpm' \
--prefix /opt/nacos \
--url '官网' \
 -m '用户名<用户邮箱>'   \
```

- `--prefix`为安装rpm后的存储目录
- `-p /home/rpm`为生成的rpm包的存储目录
- `-C /home/nacos`为nacos包目录（解压后的）

## 下载地址

nacos1.4.1 rpm包下载地址：[https://static.jiangliuhong.top/images/2021/1/nacos-1.4.1-1.x86_64.rpm](https://static.jiangliuhong.top/images/2021/1/nacos-1.4.1-1.x86_64.rpm)