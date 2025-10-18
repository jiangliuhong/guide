---
title: idea部署docker镜像
date: 2020-06-10 14:02:01
categories: 随笔
tags: 
cover: https://static.jiangliuhong.top/blogimg/other/docker-timg.jpg
---

# 使用idea打包推送docker镜像

> 本文针对maven项目

## 配置docker

#### 修改配置文件

```
vim /usr/lib/systemd/system/docker.service

在ExecStart=/usr/bin/dockerd-current 后面加上

-H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock
```

```shell
# 重载配置文件
systemctl daemon-reload
# 启动docker
systemctl start docker
```

## 配置项目

#### pom文件修改

首先在项目的`pom.xml`文件中引入依赖

```xml
<!--maven打包到docker-->
<dependency>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.33.0</version>
</dependency>
<!--maven打包到docker-->
```

然后修改`pom`文件中的打包插件，新增下面的内容

```xml
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.33.0</version>

    <configuration>
        <dockerHost>tcp://127.0.0.1:2375</dockerHost>
        <!--registry地址,用于推送镜像-->
        <registry>registry.thunisoft.com:5000</registry>
        <authConfig>
            <push>
                <username>admin</username>
                <password>123456</password>
            </push>
        </authConfig>

        <images>
            <image>
                <name>${docker.image.prefix}/${project.artifactId}:${project.version}</name>
                <alias>${project.name}</alias>
                <build>
                    <dockerFile>${project.basedir}/Dockerfile</dockerFile>
                </build>
                <run>
                    <namingStrategy>alias</namingStrategy>
                </run>
            </image>
        </images>
    </configuration>
</plugin>
```

其中，需要修改的有：

- dockerHost：修改为docker所在服务器的ip
- registry：推送镜像的地址
- authConfig：私有镜像需要设置的账号密码
- images：镜像信息
  - project.basedir：当前项目路径
  - project.name：当前项目的名称
  - name: 推送的镜像的名称，这里我使用的是变量
- dockerFile：Dockerfile文件地址

#### 编写Dockerfile

> 按照项目编写即可

有个需要注意的是，一般java项目打包成镜像，我们都会执行一个`COPY`命令，将`jar`包拷贝到镜像中，例如：

```
COPY ["./project.jar","/xx/java.jar"]
```

这里的`project.jar`必须是当前idea所在项目的机器上能找到的。例如：

```
COPY ["./target/adc.jar","/home/ROOT.jar"]
```

这里的源路径取的是`target`目录下的

## 执行

依次执行 `package`、`docker:build`、`docker:push`即可

![执行顺序](https://static.jiangliuhong.top/blogimg/other/20200610135747.png)


