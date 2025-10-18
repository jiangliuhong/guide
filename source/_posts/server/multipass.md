---
title: multipass使用文档
categories: 服务器
date: 2022-02-10 22:20:00
---

> multipass是ubuntu发布的用于快速构建ubuntu虚拟机的轻量级虚拟化软件，并且multipass同时支持windows、macos、linux平台。

## 安装multipass

### 在Linux上安装

先安装snap管理器：

- 向centos7添加EPEL

```
sudo yum install epel-release
```

centos8请使用

```
sudo dnf install epel-release
```

- 安装snap

```
sudo yum install snapd
```

- 安装后，需要启用管理主snap通信套接字的systemd单元

```
sudo systemctl enable --now snapd.socket
```

- 在/var/lib/snap/snap和/snap之间创建符号链接

```
sudo ln -s /var/lib/snapd/snap /snap
```

- 安装multipass

```
sudo snap install multipass
```

### 在Windows上安装

访问multipass官网：[https://multipass.run/](https://multipass.run/)，下载windows安装包，然后双击执行安装文件。

如果有安装`Chocolatey`(`Chocolatey`安装链接[https://chocolatey.org/install](https://chocolatey.org/install))，也可以使用一下的命令进行安装：

```
choco install multipass
```

### 在Macos上安装

访问multipass官网：[https://multipass.run/](https://multipass.run/)，下载macos安装包，然后双击执行安装文件。

通过`Homebrew`安装：

```
brew cask install multipass
```

### 验证安装结果

安装完成后，使用`multipass version`命令即可查看安装结果:

```
# multipass version
multipass   1.8.0+win
```

## 命令说明

> 更多命令说明：[https://multipass.run/docs/alias-command](https://multipass.run/docs/alias-command)

命令|说明
---|---
multipass find|查看镜像列表
multipass launch --name ubuntu1 18.04|以18.04镜像创建一个ubuntu1实例，不指定镜像时默认使用最新镜像
multipass list|查看实例列表
multipass start --all|启动所有实例
multipass stop --all|停止所有实例
multipass delete --all|删除所有实例
multipass stop ubuntu1 ubuntu2|停止ubuntu1和ubuntu2实例
multipass start ubuntu1|启动ubuntu1实例
multipass delete ubuntu1|删除ubuntu1实例
multipass info --all|查看所有实例信息
multipass info ubuntu1|查看ubuntu1实例信息
multipass shell ubuntu1|进入ubuntu1实例终端，如果该实例未启动，则会自动启动该实例
multipass purge|释放空间

launch常用参数说明（详细文档：[https://multipass.run/docs/launch-command](https://multipass.run/docs/launch-command)）：

参数|说明|示例
---|---|---
-n(--name)|指定实例名称|-n ubuntu1
-c(--cpus)|指定cpu|-c 1
-m(-mem)|指定内存|-m 1G
-d(--disk)|指定磁盘|-d 5G
--cloud-init|指定初始化配置|--cloud-init config.yaml

## 创建虚拟机

```
multipass launch --name ubuntu1 --cpus 2 --mem 2G --disk 5G --cloud-init init.config 18.04
```

使用该命令，会以`18.04`镜像创建一个实例名为ubuntu1，cpu2核，内存2G,磁盘5G的实例，同时使用`init.config`文件来初始化实例

`init.config`文件内容：

```yml
ssh_authorized_keys:
  - ssh-rsa AAAAB3.......hh32R test@163.com(生成的公钥)
runcmd:
  - sudo apt-get install python -y
```

该文件的作用为：

- 给实例增加了一个公钥，后续可通过私钥进行远程连接
    - 可以使用`ssh-keygen -t rsa -C "test@163.com"`命令生成密钥
- 实例安装完成后，默认安装python

更多关于init.config的说明请见:[https://cloudinit.readthedocs.io/en/latest/topics/examples.html](https://cloudinit.readthedocs.io/en/latest/topics/examples.html)

在实例初始化完成后，使用`multipass shell ubuntu1`命令即可进入实例的shell控制台，对实例进行操作。