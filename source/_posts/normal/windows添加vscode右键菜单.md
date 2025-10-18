---
title: windows添加vscode右键菜单
date: 2020-03-24 09:49:02
categories: 随笔
tags: 
cover: https://static.jiangliuhong.top/blogimg/tools/wondows-10-keyboard.jpg
---

# windows添加vscode右键菜单

> 最近对`vscode`编辑器几乎入魔了，凡事静态代码，都改用`vscode`来编写了，不过在windows上，如果需要打开一个文件或者一个文件夹，这个操作比较繁琐，所以就期望能够直接在windows的资源管理器里加上一个菜单。经过`google`之后，发现了如下方法

创建一个`vscode.reg`，名字可以随便写，但是后缀名用`.reg`，文件内容如下：

```
Windows Registry Editor Version 5.00
	
[HKEY_CLASSES_ROOT\*\shell\VSCode]
@="Open with Code"
"Icon"="C:\\app\\Microsoft VS Code\\Code.exe"

[HKEY_CLASSES_ROOT\*\shell\VSCode\command]
@="\"C:\\app\\Microsoft VS Code\\Code.exe\" \"%1\""

Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\Directory\shell\VSCode]
@="Open with Code"
"Icon"="C:\\app\\Microsoft VS Code\\Code.exe"

[HKEY_CLASSES_ROOT\Directory\shell\VSCode\command]
@="\"C:\\app\\Microsoft VS Code\\Code.exe\" \"%V\""

Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\Directory\Background\shell\VSCode]
@="Open with Code"
"Icon"="C:\\app\\Microsoft VS Code\\Code.exe"

[HKEY_CLASSES_ROOT\Directory\Background\shell\VSCode\command]
@="\"C:\\app\\Microsoft VS Code\\Code.exe\" \"%V\"" 
```

其中 `C:\\app\\Microsoft VS Code\\Code.exe` 这个是我本机的安装地址，你可能需要修改一下。

最后双击执行即可，如果失败，你可能需要使用管理员权限使用。
