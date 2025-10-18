---
title: Git问题总结
date: 2020-04-26 10:39:21
categories: 随笔
tags:
   - git
cover: https://static.jiangliuhong.top/blogimg/other/git-logo.png
---

## Git 问题总结

> 持续更新...

### GitHub clone慢或失败

> 2020-04-26更新

在`clone`GitHub上的仓库时，经常遇见这样的问题：

```shell
$ git clone https://github.com/jgraph/drawio.git
Cloning into 'drawio'...
remote: Enumerating objects: 634, done.
remote: Counting objects: 100% (634/634), done.
remote: Compressing objects: 100% (297/297), done.
error: RPC failed; curl 18 transfer closed with outstanding read data remaining
fatal: the remote end hung up unexpectedly
fatal: early EOF
fatal: index-pack failed
```

网络上搜索到很多解决办法，其中主要分为两点：

**1.设置代理:**

```shell
# 千万别急，刚开始而已
# socks5协议，1080端口修改成自己的本地代理端口
git config --global http.proxy socks5://127.0.0.1:1080
git config --global https.proxy socks5://127.0.0.1:1080

# http协议，1081端口修改成自己的本地代理端口
git config --global http.proxy http://127.0.0.1:1081
git config --global https.proxy https://127.0.0.1:1081
```

仅仅针对github进行配置，让github走本地代理，其他的保持不变；

```shell
# socks5协议，1080端口修改成自己的本地代理端口
git config --global http.https://github.com.proxy socks5://127.0.0.1:1080
git config --global https.https://github.com.proxy socks5://127.0.0.1:1080

# http协议，1081端口修改成自己的本地代理端口
git config --global http.https://github.com.proxy https://127.0.0.1:1081
git config --global https.https://github.com.proxy https://127.0.0.1:1081
```

重置代理：

```shell
git config --global --unset http.proxy
git config --global --unset https.proxy
```

如果不能重置的话，可以在你的操作系统中，找到`.gitconfig`文件（一般在系统用户目录下，例如windows系统是`C:\Users\Administrator\.gitconfig`），手动进行删除

**2.设置压缩:**

```shell
git config --add core.compression -1
```

当然可以直接修改 .gitconfig 文件

```
[user]
    name = Ggicci
    email = ...
[core]
    compression = -1
```

说明：

compression 是压缩的意思，从 clone 的终端输出就知道，服务器会压缩目标文件，然后传输到客户端，客户端再解压。取值为`[-1, 9]`，`-1`以`zlib`为默认压缩库，`0`表示不进行压缩，`1..9`是压缩速度与最终获得文件大小的不同程度的权衡，数字越大，压缩越慢，当然得到的文件会越小。