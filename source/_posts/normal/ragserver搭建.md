---
title: 基于DeepSeek的本地化知识库 RAGFlow 搭建
date: 2025-10-20 09:36:00:00
categories: 随笔
tags: AI
cover: https://static.jiangliuhong.top/images/picgo/20251020173608.png
---

## 部署ollama

准备镜像

```yml
docker pull ollama/ollama:0.12.6
```

启动镜像

```yml
 docker run -d --gpus=all -v /opt/ollama/data:/root/.ollama -p 11434:11434 --name ollama ollama/ollama:0.12.6
```

这里我选择直接docker run ,也可以加到ragflow的docker-compse.yml配置中

启动完成后部署**`bge-m3`**模型

```shell
docker exec -it ollama ollama pull bge-m3
docker exec -it ollama ollama pull deepseek-r1:1.5b
```

`bge-m3`模型一共1.2GB

`deepseek-r1:1.5b`模型一共1.1GB

## 部署ragflow

> 参考 https://deepseek.csdn.net/67bc6ed76670175f992a7796.html

准备配置文件

```yml
git clone https://github.com/infiniflow/ragflow.git
```

将仓库的`docker`文件复制到服务器中

需要注意一下.env中使用的ragflow镜像是`v0.21.0-slim`版，如下：

```shell
RAGFLOW_IMAGE=infiniflow/ragflow:v0.21.0-slim
```

`slim`版镜像大约7G左右，如果需要使用全量版，将镜像地址改为`v0.21.0`

```shell
RAGFLOW_IMAGE=infiniflow/ragflow:v0.21.0
```

两个版本的区别在于，`v0.21.0`包含了一些内置的嵌入模型如 `BGE` 和 `BCE`，因为内嵌了模型`v0.21.0`版本的大小会大很多，大约在30G左右

使用到的镜像列表是：

```shell
REPOSITORY            TAG                            IMAGE ID       CREATED         SIZE
infiniflow/ragflow    v0.21.0-slim                   73a672a31bbf   5 days ago      7.06GB
valkey/valkey         8                              84be4d718bb5   2 weeks ago     120MB
mysql                 8.0                            94753e67a0a9   3 weeks ago     780MB
quay.io/minio/minio   RELEASE.2025-06-13T11-33-47Z   c4260bcf2c25   3 months ago    175MB
apache/hadoop         3.4.1                          ca4486fe2816   11 months ago   2.08GB
apache/hive           4.0.1                          9bb619cde186   12 months ago   1.6GB
mysql                 8.0.39                         f5da8fc4b539   15 months ago   573MB
elasticsearch         8.11.3                         ac1eef415132   22 months ago   1.41GB
```

执行命令启动容器

```yml
docker compose up -d
```

启动完成后看见如下日志证明启动成功

```yml
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:9380
 * Running on http://172.18.0.6:9380
```

浏览器访问

```yml
http://192.168.88.135/
```

注：第一次访问需要自己注册账号

## 添加模型

登录系统后,访问模型提供商管理页面，

首先添加`chat`类型模型

![](https://static.jiangliuhong.top/images/picgo/20251020152731.png)

然后再添加`embedding`类型模型

![](https://static.jiangliuhong.top/images/picgo/20251020152218.png)

添加完成后需要将这两个模型设置为默认模型

## 创建知识库

![](https://static.jiangliuhong.top/images/picgo/20251020153103.png)

知识库创建完成后新增文件，然后点击开始按钮开始解析文件

![](https://static.jiangliuhong.top/images/picgo/20251020153446.png)

## 创建对话

在聊天页新建一个聊天

然后在聊天设置中设置一下知识库

![](https://static.jiangliuhong.top/images/picgo/20251020160115.png)

最后点击新建会话即可开始对话

![](https://static.jiangliuhong.top/images/picgo/20251020160202.png)