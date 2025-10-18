---
title: jekyllcn在windows下安装
date: 2019-05-23 09:49:02
categories: 随笔
---


## 首先安装Chocolatey

> 关于Chocolatey的安装可以查看其官网 https://chocolatey.org/install

cmd命令(管理员运行)

```
@"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
```

win10下可使用PowerShell(管理员运行)

```
Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```

值得注意的是：

在运行任何这些脚本之前，请检查<https://chocolatey.org/install.ps1>是否能够正常访问。

## 安装ruby

在成功安装Chocolatey后，我们可以使用choco来安装软件包

```
choco install ruby -version 2.2.4
choco install ruby2.devkit
```

其中 ruby2.devkit 是编译json gem时使用

配置Ruby development kitPermalink

Ruby开发工具包并没有设置Ruby环境变量，所以我们需要手动设置：

- 在`C:\tools\DevKit2`目录下打开命令行界面

- 执行`ruby dk.rb init`命令创建配置文件`config.yml`
- 编辑文件`config.yml`在其中包含Ruby路径`- C:/tools/ruby22`
- 执行命令创建路径： `ruby dk.rb install`

其中需要注意的是，如果是在win10 PowerShell中执行可能会报如下错误：

```
ruby : 无法将“ruby”项识别为 cmdlet、函数、脚本文件或可运行程序的名称。请检查名称的拼写，如果包括路径，请确保路径正确
，然后再试一次。
所在位置 行:1 字符: 1
+ ruby dk.rb install
+ ~~~~
    + CategoryInfo          : ObjectNotFound: (ruby:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
```

这是因为windows默认不允许执行任何脚本，解决办法是设置ExecutionPolicy为Unrestricted（可以运行脚本或者读取配置文件，如果执行的是从网上下载的脚本，那么会有一个申请权限的提示。 ）

```
Set-ExecutionPolicy Unrestricted
```

 当然你也可以在`CMD`中执行`ruby dk.rb install`命令

## Nokogiri软件包安装

github-pages运行时需要Nokogiri这个软件包，但是要运行在64位Windows系统上还需要执行以下命令： 

**注意:** 在当前版本 [pre release](https://github.com/sparklemotion/nokogiri/issues/1456#issuecomment-206481794) 中提供了64位Windows系统支持，但是github-pages中并没有引用这个版本。 

``` 
choco install libxml2 -Source "https://www.nuget.org/api/v2/"
choco install libxslt -Source "https://www.nuget.org/api/v2/"
choco install libiconv -Source "https://www.nuget.org/api/v2/"
```

然后找到通过choco安装的三个包的路径，我的电脑安装在`C:\ProgramData\chocolatey\lib`下

```
c:\tools\DevKit2>gem install nokogiri --^
 --with-xml2-include=C:\ProgramData\chocolatey\lib\libxml2\build\native\include^
 --with-xml2-lib=C:\ProgramData\chocolatey\lib\libxml2.redist\build\native\bin\v110\x64\Release\dynamic\cdecl^
 --with-iconv-include=C:\ProgramData\chocolatey\lib\libiconv\build\native\include^
 --with-iconv-lib=C:\ProgramData\chocolatey\lib\libiconv.redist\build\native\bin\v110\x64\Release\dynamic\cdecl^
 --with-xslt-include=C:\ProgramData\chocolatey\lib\libxslt\build\native\include^
 --with-xslt-lib=C:\ProgramData\chocolatey\lib\libxslt.redist\build\native\bin\v110\x64\Release\dynamic
```



## 安装 github-pagesPermalink

- 打开命令行界面安装 [Bundler](http://bundler.io/): `gem install bundler`
- 在你的博客根目录中创建名为 `Gemfile` 不带任何后缀名的文件
- 拷贝复制下面两行到文件中：

```
source 'http://rubygems.org'
gem 'github-pages'
```

- **注意:** 由于在使用的Ruby版本中使用SSL链接报错，所以这里我们使用不加密的链接
- 打开命令行界面，切换到你本地博客库的根目录，安装github-pages: `bundle install`

这个过程完成之后你应该就已经在系统上安装了github-pages，此时你可以通过 `jekyll s` 命令来在本地启动你的博客。 
在启动的过程你会得到一个警告信息，提示你应该在 `Gemfile` 中包含 `gem 'wdm', '>= 0.1.0' if Gem.win_platform?`， 但是我在文件中添加了这一行之后 `jekyll s` 就不能正常启动了，所以我就直接无视了这个警告。

将来github-pages的安装应该像安装博客一样的简单，但是目前 Nokogiri ([v1.6.8](https://github.com/sparklemotion/nokogiri/releases)) 的最新版并不是稳定版本，没有在github-pages中应用，故我们在Windows上还是要手动安装配置。