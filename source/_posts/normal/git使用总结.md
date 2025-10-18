---
title: git使用总结
date: 2020-04-26 10:24:31
categories: 随笔
tags:
   - git
cover: https://static.jiangliuhong.top/blogimg/other/git-logo.png
---

# git使用总结

## 安装GIT


下载地址：[https://git-scm.com/downloads](https://git-scm.com/downloads)


## 网络文档、教程


官方中文教程地址：[https://git-scm.com/book/zh/v2](https://git-scm.com/book/zh/)


廖雪峰的官网GIT教程：[https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

## 配置global

git config --list 查看配置

git config --global 增加配置

```
$ git config --global user.name testname
$ git config --global user.email test@example.com
$ git config --global gui.encoding utf-8
```

一般情况，在安装git之后，都会指定name、email以及编码。

对于--global参数为针对所有项目，如果只针对一个项目设置，则去除该参数。

## Git Bash

### 打开方式

windows系统：鼠标右键，点击Git Bash Here

### 命令简介

- 克隆项目： git clone 项目地址
- 将目前使用的推送至缓存空间：git add
- 提交项目至本地仓库： git commit
- 从远程仓库更新项目： git pull
- 推送项目至远程仓库： git push
- 检出项目： git checkout
- 查看分支： git branch

### 流程介绍

#### 克隆项目

```
$ git clone 项目地址
$ cd 项目文件夹
$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/master
```

其中带*号的为当前分支，origin的为远程分支

对于开发而言，一般Master分支为稳定分支，开发一般在dev分支下进行，dev分支为自己新建的分支。

切换分支的命令：git checkout dev ，git checkout -b dev 。参数-b的作用为，如果dev分支不存在，则新建一个dev分支。

#### 提交项目至本地仓库

在修改项目后，首先需要使用git status 查看当前文件状态，然后使用git add 命令，将修改提交至本地缓冲区，再使用git commit 提交至本地仓库。

添加所有：git add . 

添加指定文件：git add 文件路径 

commit 之后，使用命令gitk可查看当前分支状况。

在修改文件后，可使用git chekout .（还原所有） 或 git checkout 文件路径（还原指定文件） 进行还原。

撤销git add操作

撤销所有的已经add的文件: git reset HEAD .

如果是撤销某个文件或文件夹：git reset HEAD -文件名称路径

```
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        test20180103.txt

nothing added to commit but untracked files present (use "git add" to track)

$ git add .
warning: LF will be replaced by CRLF in test20180103.txt.
The file will have its original line endings in your working directory.

$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        new file:   test20180103.txt

$ git commit -m "add test20180103"
[master f4859ee] add test20180103
 1 file changed, 1 insertion(+)
 create mode 100644 test20180103.txt

$gitk &

```

#### 提交项目至远程仓库

提交到远程仓库时，应先pull远程仓库内容，在pull时，可添加参数--rebase进行分支合并，然后再push到远程仓库。

在pull --rebase时可能出现冲突，解决办法见下文。

```
$ git pull --rebase
Current branch master is up to date.

$ git push
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 349 bytes | 349.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://gitee.com/ja_rome/test.git
   5a60929..f4859ee  master -> master
```

#### 解决冲突

> 对于冲突的解决，互联网上的例子有很多，此处只对meld进行介绍

##### windows系统安装meld

下载meld：[http://meldmerge.org/](http://meldmerge.org/)

下载git-diffall： [http://download.csdn.net/download/jlh912008548/9791481](http://download.csdn.net/download/jlh912008548/9791481)

安装meld

将安装目录添加至path环境变量中，如 d:/tool/meld

在git bash 中配置

```
git config --global alias.diffall "/D/tool/git-diffall"
git config --global mergetool.meld.path "/D/tool/Meld/Meld.exe"
git config --global mergetool.keepBackup false 
```
##### 解决冲突所涉及的命令

```
//新建文件，内容为123456，并提交至本地仓库
$ vim testm.txt
$ git add .
$ git commit -m "add testm"
//模拟冲突，新建tm分支
$ git checkout -b tm
//修改testm.txt内容为1234567890
$ git add .
$ git commit -m "update testm"
//回到master分支修改文件
$ git checkout master
$ vim testm.txt
//修改内容为123456123456
$ git add .
$ git commit -m "update testm"
//合并分支
$ git merge --no-ff -m "merge tm" tm
Auto-merging testm.txt
CONFLICT (content): Merge conflict in testm.txt
Automatic merge failed; fix conflicts and then commit the result.
//CONFLICT 为冲突的标记，pull --rebase的效果与之相同
//解决冲突
$ git mergetool
//此时会弹出meld的窗口，如图一所示
//将下图所有内容修改为1234567890123456
//如果是分支的合并时发生的重涂
$ git add .
$ git commit -m ""
//如果pull --rebase时发生的冲突
$ git add .
$ git rebase --continue
$ git push
```

解决冲突预览图：

![meld-view](https://static.jiangliuhong.top/blogimg/other/git-meld-view.jpg)

## Git 可视化工具

推荐使用github推出的`GitHub Desktop`工具，下载地址[https://desktop.github.com/](https://desktop.github.com/)

使用截图：

![GitHub Desktop](https://desktop.github.com/images/github-desktop-screenshot-windows.png)