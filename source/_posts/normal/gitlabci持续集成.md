---
title: GitLab持续集成环境搭建
date: 2019-03-26 00:17:00:00
categories: 随笔
tags:
   - git
---

# GitLab持续集成环境搭建

> 随着微服务的兴起，部署服务越来越繁琐，当然业内也有对应的持续集成方案，目前在企业中使用较为广泛的持续集成方案应是gitlab持续集成、持续部署（gitlab CI/CD）

## 安装GitLab与GitLab-Runner

> 为了简化安装，本文使用docker进行操作

**GitLab安装：**

首先拉去镜像：

```shell
$ docker pull gitlab/gitlab-ce
```

拉取成功后使用如下命令启动容器：

```shell
docker run -itd -p 8443:443 -p 8081:8081 -p 2222:22 \
--name gitlab --restart always \
--volume ~/develop/docker/gitlab/config:/etc/gitlab \
--volume ~/develop/docker/gitlab/logs:/var/log/gitlab \
--volume ~/develop/docker/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:latest
```

需要注意的是：

- 端口`8443` ,`8081`,`2222`可任意选择一个物理机上可用的端口。
- 映射内部`8081`端口的原因是，如果直接启动，在拉取代码的时候，网页会返回对应的容器id作为拉取的url，例如：`http://1fj3280sd/jlh/test`,对于这种情况，只需在`--volume ~/develop/docker/gitlab/config`中找到`gitlab.rb`文件，修改其中的配置项即可，`external_url "http://ip:8081"`。但是如果直接设置该属性，会使得容器内gitlab端口改为8081，会导致端口映射失败，顾此处要暴露8081端口。
- `—volume`是将容器中的地址映射到物理机上。
- 第一次启动容器，由于要进行初始化，所以启动时间较慢。

**GitLabRunner安装：**

拉取镜像：

```shell
$ docker pull gitlab/gitlab-runner
```

启动gitlab-runner容器：

```shell
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

## 注册Runner

### 容器之间互联

在注册Runner的时候需要输入gitlab的访问地址，而docker容器之间不能直接连接，所以需要进如下操作使得容器互联。

容器之间的互联有三种方式：

- 通过容器ip直接连接

使用命令`apt-get install iputils-ping`、`apt-get install net-tools`安装`ping`与`ifconfig`，安装完成之后查看容器ip

- 运行容器的时候加上`--link 容器名称:容器别名`，此时可通过容器的别名进行连接
- 创建bridge网络

使用`docker network create testnet`创建网络，然后在运行容器的时候使用该网络，使用方法：

docker run -it --name <容器名> ---network <bridge> --network-alias <网络别名> <镜像名>

此处我选用的直接用ip连接。

查看容器的两种方法：

- 使用`docker exex -it 容器id /bin/bash`进去容器，安装`ifconfig`，并使用该命令查看容器ip
- 安装docker控制台工具，在控制工具中可直接查看容器的ip等信息，我使用的是`portainer`，镜像网易云链接：[https://c.163yun.com/hub#/m/repository/?repoId=78662](https://c.163yun.com/hub#/m/repository/?repoId=78662)

### 注册

首先进入容器，使用`gitlab-runner register`命令注册：

```shell
root@f3c47cad0bf8:/# gitlab-runner register
Runtime platform                                    arch=amd64 os=linux pid=82 revision=6946bae7 version=12.0.0
Running in system-mode.                            
                                                   
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://172.17.0.3:8081 #gitlab访问地址
Please enter the gitlab-ci token for this runner:
cFa_ASWeispNJRBxs7Uz #gitlab仓库注册令牌
Please enter the gitlab-ci description for this runner:
[f3c47cad0bf8]: test 
Please enter the gitlab-ci tags for this runner (comma separated):
test
Registering runner... succeeded                     runner=cFa_ASWe
Please enter the executor: docker-ssh+machine, kubernetes, docker, parallels, shell, ssh, virtualbox, docker+machine, docker-ssh:
docker
Please enter the default Docker image (e.g. ruby:2.6):
alpine:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
root@f3c47cad0bf8:/# gitlab-runner start #启动容器
Runtime platform                                    arch=amd64 os=linux pid=90 revision=6946bae7 version=12.0.0
root@f3c47cad0bf8:/# 
```

gitlab仓库注册令牌获取方式如图：

![令牌获取方式](https://static.jiangliuhong.top/blogimg/gitlab/token.jpg)

在完成注册后，在gitlab里可以查看到如下内容：

![runner注册成功预览](https://static.jiangliuhong.top/blogimg/gitlab/runner-one-success.jpg)



### 注册群组runner

注册群主runner与单个方式相同，首先在gitlab上新建一个群组，然后在该群组的[设置->CI/CD->Runner]中查看Runner注册令牌，然后在注册runner的时候使用该令牌即可。

