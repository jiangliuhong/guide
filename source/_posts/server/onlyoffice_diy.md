---
title: onlyoffice入门使用
date: 2020-07-20 22:37:56
categories:
    - 服务器
tags:
    - onlyoffice
cover: https://static.jiangliuhong.top/blogimg/other/onlyoffice-jj.jpg
---

## onlyoffice安装

采用`docker`进行安装，命令为：

```
#下载镜像
docker pull onlyoffice/documentserver:5.5
# 启动镜像
docker run -itd -p 8009:80 onlyoffice/documentserver:5.5
```

访问`127.0.0.1:8009`

## onlyoffice汉化

1. 删除容器里的文件，替换windows下的字体。

使用`docker exec -it 容器id /bin/bash`进入容器。

删除`X11`文件夹、删除`dejavu`

```
rm -rf /usr/share/fonts/X11
rm -rf /usr/share/fonts/truetype/dejavu
```

执行`exit`命令退出容器

准备好字体文件，访问[https://github.com/jiangliuhong/font-zh.git](https://github.com/jiangliuhong/font-zh.git)，执行命令:

```
# 拉取仓库
git clone https://github.com/jiangliuhong/font-zh.git
cd font-zh/font
# 拷贝字体到容器中
tar -cv*|docker exec -i 容器ID tar x -C /usr/share/fonts/truetype
```

进入容器

```
docker exec -it 容器ID /bin/bash
```
依次执行一下命令：

```
sudo mkfontscale
sudo mkfontdir
sudo fc-cache -fv
```

执行`exit`命令退出容器

运行`documentserver-generate-allfonts.sh`然后清理浏览器缓存

```
docker exec 容器ID /usr/bin/documentserver-generate-allfonts.sh
```

编写页面也是是否修改成功，其中需要在editorConfig添加语言配置：

```
"lang": "zh-CN"
```

html文件：
```html
<html>
<div id="placeholder"></div>
<script type="text/javascript" src="http://127.0.0.1:8009/web-apps/apps/api/documents/api.js"></script>
<script>
    var config = {
        "document": {
            "fileType": "docx",
            "key": "Khirz6zTPdfd7",
            "title": "Example Document Title.docx",
            "url": "http://blog.jiangliuhong.top/blogimg/other/test.docx"
        },
        "documentType": "text",
        "editorConfig": {
            "callbackUrl": "https://example.com/url-to-callback.ashx",
            "user": {  
                        "id": "{{.Uid}}",  
                        "name": "{{.Uname}}"  
                    }, 
            "lang": "zh-CN"
        }
    };
    var docEditor = new DocsAPI.DocEditor("placeholder", config);
</script>

</html>
```

## 导出自己的镜像

在上一步中对容器内的onlyoffice进行了中文汉化，但是如果此时删除容器，再创建容器，则需要在进行一次操作，此时就需要把容器导出为镜像，命令如下：

```
docker commit 容器id 镜像名:tag
# 例如
docker commit f6e9c9700c93 onlyoffice/documentserver:5.5_zh
```

导入镜像后，镜像目前仅存在本机电脑而已，可以将其上传到公司的镜像服务器或者国内的阿里、网易的镜像服务器，甚至于docker.io上，由于国内网络限制，我选择了上传到阿里云上。

首先前往阿里云容器镜像服务中创建命令空间，然后在访问凭证中设置登陆凭证，然后在执行登陆的命令，详细如图：

![aliyun_docker_push](https://static.jiangliuhong.top/blogimg/other/aliyun_docker_push.png)

对进行进行重命名（复制镜像）:

```
docker tag onlyoffice/documentserver:5.5_zh registry.us-west-1.aliyuncs.com/(阿里云上创建的命名空间)/onlyoffice/documentserver:5.5_zh
```

执行push命令:

```
docker push registry.us-west-1.aliyuncs.com/(阿里云上创建的命名空间)/onlyoffice/documentserver:5.5_zh
```
