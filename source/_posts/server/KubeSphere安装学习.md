---
title: KubeSphere安装学习
categories: 服务器
date: 2021-03-30 23:55:07
cover: https://static.jiangliuhong.top/images/2021/3/1617117940367.png
---

## 前期准备

准备三台虚拟机，系统为centos7，内存至少分配2G，虚拟机如下：

IP|HOST|角色
---|---|---
192.168.152.131|master.myk8s.com|master
192.168.152.132|node1.myk8s.com|node
192.168.152.133|node2.myk8s.com|node

分别修改三台机器的HOST(`/etc/hosts`)，内容如下

```
192.168.152.131 master.myk8s.com
192.168.152.132 node1.myk8s.com
192.168.152.133 node2.myk8s.com
```

为了避免防火墙导致服务无法访问，先关闭防火墙

- systemctl stop firewalld.service
- systemctl disable firewalld.service

关闭selinux,编辑文件`/etc/selinux/config`，将SELINUX=enforcing改为SELINUX=disabled，修改该操作需要重启


## 安装docker

> 可以使用`KubeKey`安装，也可自己进行安装，为避免镜像下载速度导致安装过程缓慢，最好是先自行安装了docker后再安装KubeSphere
> 
> 下面简单说明下离线安装方法


首先前往docker官网下载离线安装包：[https://download.docker.com/linux/static/stable/](https://download.docker.com/linux/static/stable/)

这里我下载的是x86_64的20.10.5版本，下载链接为：[https://download.docker.com/linux/static/stable/x86_64/docker-20.10.5.tgz](https://download.docker.com/linux/static/stable/x86_64/docker-20.10.5.tgz)
### 准备三个脚本

> 如果在windows上编辑过，在服务器上可能会报这样的错误：` bad interpreter: No such file or directory`，这个错误是windows上的换行符导致的，使用vscode打开，将换行标识改为`LF`即可。

**docker.service**

```bash
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```


**install.sh**

```bash
#!/bin/bash
echo '解压tar包...'
tar -xvf $1

echo '将docker目录移到/usr/bin目录下...'
cp docker/* /usr/bin/

echo '将docker.service 移到/etc/systemd/system/ 目录...'
cp docker.service /etc/systemd/system/

echo '添加文件权限...'
chmod +x /etc/systemd/system/docker.service

echo '重新加载配置文件...'
systemctl daemon-reload

echo '启动docker...'
systemctl start docker

echo '设置开机自启...'
systemctl enable docker.service

echo 'docker安装成功...'
docker -v
```

**uninstall.sh**

```bash
#!/bin/bash
echo '删除docker.service...'
rm -f /etc/systemd/system/docker.service

echo '删除docker文件...'
rm -rf /usr/bin/docker*

echo '重新加载配置文件'
systemctl daemon-reload

echo '卸载成功...'
```

### 执行安装

将docker的安装包以及上面准备的脚本上传到服务器上，然后执行`install.sh`脚本即可，这里我也准备了一个包含脚本的docker-20.10.5的安装包，下载地址为：[https://static.jiangliuhong.top/tools/docker-20.10.5-install.tar.gz](https://static.jiangliuhong.top/tools/docker-20.10.5-install.tar.gz)，执行示例如下：

```bash
./install.sh docker-20.10.5.tgz
```

安装脚本执行完后，执行下面的检验命令，成功返回则代表安装成功：

```shell
[root@master docker-20.10.5-install]# docker -v
Docker version 20.10.5, build 55c4c88
[root@master docker-20.10.5-install]# docker images
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE
[root@master docker-20.10.5-install]#
```

### 设置镜像加速器

在安装时，安装程序会从镜像仓库中下载镜像，如果使用默认的镜像源，则会导致下载时间较长，这里推荐设置阿里云的镜像加速器

首先从阿里云的容器镜像服务->镜像工具->镜像加速器中获取加速器地址

![](https://static.jiangliuhong.top/images/2021/3/1616681160094.png)

修改`daemon.json`文件

```json
{
  "registry-mirrors": ["加速器地址"]
}
```

修改之后执行命令：

```shell
sudo systemctl daemon-reload 
sudo systemctl restart docker
```

## 在Linux上以All-in-One模式安装KubeSphere

> 官网参考地址：[https://kubesphere.com.cn/docs/quick-start/all-in-one-on-linux/](https://kubesphere.com.cn/docs/quick-start/all-in-one-on-linux/)

先给服务器安装必要的组件，按照官网的描述，需要安装`
socat`、`conntrack`、`ebtables`、`ipset`，具体安装命令为：

```shell
yum install -y socat
yum install -y conntrack
yum install -y ebtables
yum install -y ipset
```

下载`KubeKey`安装包，官方推荐的是直接使用下面的命令：

```shell
curl -sfL https://get-kk.kubesphere.io | VERSION=v1.0.1 sh -
```

但是我在虚拟机上总是下载失败，于是从`github`上下载安装包，下载地址为：[https://github.com/kubesphere/kubekey/releases](https://github.com/kubesphere/kubekey/releases),如果你的电脑访问github非常慢的话，可以使用我的下载地址：[https://static.jiangliuhong.top/tools/kubekey-v1.1.0-alpha.1-linux-amd64.tar.gz](https://static.jiangliuhong.top/tools/kubekey-v1.1.0-alpha.1-linux-amd64.tar.gz)

在执行安装命令之前，为防止storage.googleapis.com访问受限导致安装失败，需要先执行一下下面的命令：

```bash
export KKZONE=cn
```

下载完成后，执行安装命令：

```shell
./kk create cluster --with-kubernetes v1.17.9 --with-kubesphere v3.0.0
```

执行命令后，会有一个软件校验，输入一个yes即可

![](https://static.jiangliuhong.top/images/2021/3/1616753136988.png)

安装程序会自动下载所依赖的安装包以及所需要的镜像，等待安装完成即可。


在安装完成后执行一下命令可校验安装结果

```shell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```

执行结果为：

```shell
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://192.168.0.2:30880
Account: admin
Password: P@88w0rd

NOTES：
  1. After logging into the console, please check the
     monitoring status of service components in
     the "Cluster Management". If any service is not
     ready, please wait patiently until all components
     are ready.
  2. Please modify the default password after login.

#####################################################
https://kubesphere.io             20xx-xx-xx xx:xx:xx
#####################################################
```

按照指示访问控制台即可。

## 卸载

下面是卸载单机版的命令

```shell
./kk delete cluster
```

下面是卸载集群版的命令

```shell
./kk delete cluster [-f config-sample.yaml]
```

执行kk命令的卸载后，docker仍会有残余镜像与容器，如果要删除的话，可执行下面的命令，注意，请谨慎执行：

```shell
#停止所有容器
docker stop $(docker ps -aq)
#删除所有容器
docker rm $(docker ps -aq)
#删除所有镜像
docker rmi $(docker images -q)
```

## 集群安装

> TODO