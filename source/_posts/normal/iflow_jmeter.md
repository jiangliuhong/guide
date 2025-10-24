---
title: 使用iflow进行压力测试
date: 2025-10-24 15:50:45
categories:
  - 随笔
tags: AI
cover: https://static.jiangliuhong.top/images/picgo/20251024160651.png
---
如果你使用的是windows系统，特别建议安装一个Ubuntu的WSL进行 iflow的使用

以下我是基于 Windows11 完成的

## 安装iflow

官方提供了很方便的一键安装脚本,[跳转官网教程](https://platform.iflow.cn/cli/quickstart#%E5%BF%AB%E9%80%9F%E5%AE%89%E8%A3%85)

```
bash -c "$(curl -fsSL https://gitee.com/iflow-ai/iflow-cli/raw/main/install.sh)"
```

但是window环境安装稍微麻烦一点,官方说明如下：

```
1. 访问 https://nodejs.org/zh-cn/download 下载最新的 Node.js 安装程序
2. 运行安装程序来安装 Node.js
3. 重启终端：CMD(Windows + r 输入cmd) 或 PowerShell
4. 运行 `npm install -g @iflow-ai/iflow-cli@latest` 来安装 iFlow CLI
5. 运行 `iflow` 来启动 iFlow CLI
```

## 安装MCP

官方（[https://platform.iflow.cn/mcp](https://platform.iflow.cn/mcp)）的只给出了在linux/macos上jmeter mcp应用的方法，如图所示

![image](https://static.jiangliuhong.top/images/picgo/20251024103019.png)

如果是linxu\macos或者是windwos WSL系统可直接使用官网的命令进行安装

```
iflow mcp add-json -s user 'jmeter-mcp-server' "{\"command\":\"uvx\",\"args\":[\"--from\",\"iflow-mcp_jmeter-mcp-server\",\"jmeter-mcp-server\"],\"env\":{\"JMETER_HOME\":\"/opt/app/apache-jmeter-5.6.3\",\"JMETER_BIN\":\"${JMETER_HOME}/bin/jmeter\",\"JMETER_JAVA_OPTS\":\"-Xms1g -Xmx2g\"}}" 
```

如果你是windows需要简单转换一下（当然不建议直接使用widnows）

```shell
iflow mcp add-json --% jmeter-mcp-server '{"command":"uvx","args":["--from","iflow-mcp_jmeter-mcp-server","jmeter-mcp-server"],"env":{"JMETER_HOME":"C:\\Workspace\\app\\apache-jmeter-5.6.3","JMETER_BIN":"${JMETER_HOME}\\bin\\jmeter.bat","JMETER_JAVA_OPTS":"-Xms1g -Xmx2g"}}'
```

但是可能是存在windows兼容性问题，安装总出错：

```shell
(Use node --trace-deprecation ... to show where the warning was created) 用法：gemini mcp add-json [选项] <名称> <json> Positionals: name 服务器名称 [string] [required] json MCP 服务器的 JSON 配置字符串 [string] [required] Options: -s, --scope 配置范围（用户或项目） [string] [choices: "user", "project"] [default: "project"] -h, --help Show help [boolean] Not enough non-option arguments: got 0, need at least 2
```

于是切换为手动安装的方法

找到文件`C:\Users\[windows用户]\.iflow\settings.json`

然后添加如下信息：(注意需要把`JMETER_HOME`、`JMETER_BIN`改为你的实际参数)

```json
"mcpServers": {
    "jmeter-mcp-server": {
      "command": "uvx",
      "args": [
        "--from",
        "iflow-mcp_jmeter-mcp-server",
        "jmeter-mcp-server"
      ],
      "env": {
        "JMETER_HOME": "C:\\Workspace\app\\apache-jmeter-5.6.3",
        "JMETER_BIN": "C:\\Workspace\app\\apache-jmeter-5.6.3\\bin\\jmeter.bat",
        "JMETER_JAVA_OPTS": "-Xms1g -Xmx2g"
      }
    }
  }
```

配置完成后，执行以下命令查看安装情况

```shell
iflow mcp list
```

提示

```shell
jmeter-mcp-server: uvx --from iflow-mcp_jmeter-mcp-server jmeter-mcp-server (stdio) - 已断开
```

导致这个可能的原因是

1. uvx 名称不存在，解决办法是安装一下：`pip install uvx`，也可以指定源安装`pip install uv -i https://pypi.tuna.tsinghua.edu.cn/simple`
2. `uvx --from iflow-mcp_jmeter-mcp-server jmeter-mcp-server`超时了，可以先手动执行这个命令初始化一下，同时也可以加一个代理提速：`uvx --proxy http://127.0.0.1:7890 --from iflow-mcp_jmeter-mcp-server jmeter-mcp-server`

如果没有问题执行`iflow mcp list`将会出现如下提示：

```shell
jmeter-mcp-server: uvx --from iflow-mcp_jmeter-mcp-server jmeter-mcp-server (stdio) - 已连接
```


## 初始化项目

新建一个jmeter的工作目录：`C:\Workspace\localproject\jmeter`，在该目录打开终端执行`iflow`，然后执行`/init`进行初始化，等待初始化完成后输入指令让`iflow`创建文件，当然也可以自己再`jmeter`中进行创建：

```json
这是一个执行jmeter测试的项目，现在创建一个test.jmx文件
```

有了jmx文件后，就可以让`iflow`帮我们编写测试方案了，我输入的指令如下：

```
项目的服务地址是：endpoint -> http://127.0.0.1:8001
登录接口是：
POST ${endpoint}/api/base-oauth/oauth/login?username=jlh_api&password=fa41cf1caf80ee26ea817f0c447da34c HTTP/1.1
Accept: application/json
Content-Type: application/json
登录接口返回的Body结构是：{"flag":true,"code":null,"message":null,"data":{"access_token":"xxxx"}}
其中flag为true时代表登录成功，data.access_token的值为token
对于其他接口传递token的方式是，添加一个header，Authorization:${token}
当前用户接口是:
GET ${endpoint}/api/base-user/api/v1/organ/currentUser
Accept: application/json
Content-Type: application/json
Authorization:${token}
现在需要对当前用户接口进行压测，把内容写入到test.jmx文件
```

在`iflow`执行过程中会打印它本次执行的事项，如下所示

```
 ·已更新待办事项列表                                                                                
 │      ⎿ ☐ 修改 test.jmx 文件以实现登录并获取 token                                                     
 │        ☐ 在 test.jmx 中添加对当前用户接口的压测                                                       
 │        ☐ 配置线程组和循环控制器以进行压力测试  
```

等待`iflow`执行完成后，打印提示如下：

```
✦ 好的，我已经完成了您的要求。
  1.  我修改了 test.jmx 文件，添加了登录请求和提取 access_token 的步骤。
  2.  我添加了对当前用户接口的压测请求，并配置了 Authorization header 来传递 token。
  3.  我配置了线程组和循环控制器以进行压力测试：
      *   线程数设置为 10
      *   Ramp-Up 时间设置为 5 秒
      *   测试持续时间设置为 60 秒
      *   循环次数设置为永远循环（-1）
```

打开`jmeter`查看效果如下

![image](https://static.jiangliuhong.top/images/picgo/20251024112416.png)

这时我再jmeter执行脚本发现在执行当前用户接口的时候没有加上token，于是继续提问

```
当前用户接口的header没有绑定登录接口返回的token
```

![image](https://static.jiangliuhong.top/images/picgo/20251024112845.png)

接着执行测试，输入`/mcp list `

![image](https://static.jiangliuhong.top/images/picgo/20251024114225.png)

可以看到，之前我们安装的mcp服务提供了好几个方法，这里选择使用 `execute_jmeter_test`执行测试

```
execute_jmeter_test test.jmx  
```

运行结果为

```
✦ The JMeter test has been executed successfully. Here's a summary of the results:

   - Total samples: 998
   - Average response time: 630 ms
   - Minimum response time: 40 ms
   - Maximum response time: 1650 ms
   - Error rate: 0.00%
   - Throughput: ~16.5 samples/second

  The test ran for approximately 1 minute. No errors were encountered during the execution.
```