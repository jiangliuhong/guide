---
title: go gRPC学习
categories: 
    - golang
date: 2023-08-26 10:52:08
tags:
    - golang
    - gRPC
cover: https://static.jiangliuhong.top/images/picgo/20251018224126.png
---

## gRPC简介

gRPC是由google开源的一个高性能的RPC框架。它是由Google内部的RPC框架演化而来的，于2015年正式开源，是云原生时代的一个RPC标准。

gRPC的核心设计思路

- 网络通信：gRPC自己缝状的网络通信，提供多种语言的网络通信的封装，支持的语言有:C、Java、GO
- 协议：基于HTTP2协议，使用二进制数据内容传输，支持双向流（双工）连接的多路复用
- 序列化：基于二进制，使用protobuf进行序列（protobuf是google开源的一种序列化方式，时间、空间效率是JSON的3～5倍）
- 代理的创建：让调用者像调用本地方法一样去调用远端服务

解释一下HTTP2：
- 一个效率高于HTTP1的二进制协议
- 多路复用：一个请求（连接）可以请求多个数据
- 三个新的概念
  - 数据流
    - 可以通过设置权重限制不同流的传输顺序
    - 流控，服务端可以暂停客户端流的发送
  - 消息
  - 帧

![1693062598088.png](https://static.jiangliuhong.top/images/2023/8/1693062598088.png)

gRPC与ThriftRPC区别：

- 相同点：支持异构语言的RPC
- 区别：
  - 网络通信
    - ThriftRPC使用的是Thrift 专属的TCP协议
    - gRPC使用HTTP2
  - 性能:ThriftRPC效率高于gRPC
  - gRPC大厂背书（google），云原生时代优势

gRPC的好处：

- 高效的进行进程间通信
- 支持多种语言，对C、Go、Java提供原生支持（其他语言可以C语言为基础进行扩展，例如C++、C#、Python等...）
- 支持多平台运行：Linux、Android、IOS、MacOS、Windows
- gRPC序列化方式（protobuf）效率高
- 使用HTTP2协议

## protobuf (Protocol Buffers)

protobuf是一种与编程语言无关的(IDL)，与具体平台无关的(OS)中间语言，可以方便client于server中进行RPC的数据传输。

protobuf目前有两种版本，分别为proto2与proto3，但目前主流应用使用的是proto3。

使用protobuf时需要安装protobuf的编译器，通过编译器把protobuf的IDL语言转换为一种具体的开发语言。

如何安装编译器？

```
brew install protobuf
```

其他系统可以在github上进行下载[https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases)

基本语法：

- 文件以`.proto`结尾，文件中除结构定义以外需要以分号结尾，`service`中的`rpc`方法结尾分号可以省略
- 结构定义有`message`、`service`、`enum`
- 结构名词需要使用驼峰命名方式，`message`字段使用小写字母加下划线的方式，`enum`字段使用大写字母加下划线的方式
- `service`中定义的方法使用驼峰命名

proto文件定义
```
syntax = "proto3";
package user;
option go_package = "./;protos";

```

syntax: 声明protobuf版本
package: 生成文件的包
go_package: 生成的go文件的包

定义一个枚举（`enum`）:
```
enum UserType {
  USER_TYPE_UNSPECIFIED = 0;
  USER_TYPE_ADMIN = 1;
  USER_TYPE_PORTAL = 2;
}
```

定义一个消息（`message`）
```
message UserInfo{
  string username = 1;
  string password = 2;
  UserType type = 3;
  repeated string roles = 4;
}
```
注意字段后面的数字为占位符，不可重复，常用的protobuf字段类型有：

- bool 布尔
- double 64位浮点数
- float 32位符点数
- int32 32位整数
- int64 64位整数
- uint32 无符号32位整数
- uint64 无符号64位整数
- sint32 32位整数，处理负数效率高
- sint64 64位整数，处理负数效率高
- string 字符串
- bytes 字节数组

repeated 关键字用于声明数组

`message`中嵌套

```
message UserSearchResponse{
  int32 code = 1;
  string message = 2;
  UserInfo userInfo = 3;
}
```

定义一个服务（`service`）
```
service UserService {
  rpc Search(UserSearchRequest) returns (UserSearchResponse){}
}
```

更多语法见官网：[https://protobuf.dev/](https://protobuf.dev/)


## 在go项目中使用grpc

> 示例源码：[https://github.com/jiangliuhong/gostudy/tree/main/grpc-study](https://github.com/jiangliuhong/gostudy/tree/main/grpc-study)

首先安装protobuf的go代码生成器

```
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```


首先在项目中引入grpc需要的依赖:

```
go get -u google.golang.org/grpc
go get -u google.golang.org/protobuf
```

然后分别创建两个目录，一个server(用来放服务端代码)一个client(用来放客户端代码)，在server目录下定义user.proto文件，文件内容为：

```
syntax = "proto3";

option go_package = "./;protos";

enum UserType {
  USER_TYPE_UNSPECIFIED = 0;
  USER_TYPE_ADMIN = 1;
  USER_TYPE_PORTAL = 2;
}

message UserSearchRequest {
  string username = 1;
}

message UserInfo{
  string username = 1;
  string password = 2;
  UserType type = 3;
  repeated string roles = 4;
}

message UserSearchResponse{
  int32 code = 1;
  string message = 2;
  UserInfo userInfo = 3;
}


service UserService {
  rpc Search(UserSearchRequest) returns (UserSearchResponse){}
}
```

在该目录执行生成命令

```
# 生成protobuf定义的结构
protoc --go_out=. user.proto
# 生成grpc需要的代码
protoc --go-grpc_out=. user.proto
```

代码生成完成后，将生成的`user.pb.go`文件拷贝到客户端目录下

![](https://static.jiangliuhong.top/images/2023/9/1694100142308.png)

编写`server/main.go`文件

```go
package main

import (
	"context"
	"google.golang.org/grpc"
	"grpc-study/server/protos"
	"net"
)

// userService 实现grpc服务端结构体
type userService struct {
	protos.UnimplementedUserServiceServer
}

// Search 实现grpc定义的rpc方法
func (u *userService) Search(ctx context.Context, usr *protos.UserSearchRequest) (*protos.UserSearchResponse, error) {
	username := usr.Username
	// 模拟输出
	return &protos.UserSearchResponse{
		Code:    1,
		Message: "success",
		UserInfo: &protos.UserInfo{
			Username: username,
			Password: "xxx",
			Type:     protos.UserType_USER_TYPE_ADMIN,
		},
	}, nil
}

func main() {
	// 定义grpc服务监听的端口
	listen, err := net.Listen("tcp", ":9091")
	if err != nil {
		panic(err)
	}
	// 初始化一个grpc服务
	grpcServer := grpc.NewServer()
	// 将定义的userService服务注册到grpc服务中
	protos.RegisterUserServiceServer(grpcServer, &userService{})
	err = grpcServer.Serve(listen)
	if err != nil {
		panic(err)
	}
}

```

编写`client/main.go`文件
```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"grpc-study/server/protos"
)

func main() {
	// 采用匿名的方式连接到grpc服务端
	conn, err := grpc.Dial("127.0.0.1:9091", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		panic(err)
	}
	defer conn.Close()
	client := protos.NewUserServiceClient(conn)
	// 执行grpc方法
	request := protos.UserSearchRequest{Username: "test"}
	search, err := client.Search(context.Background(), &request)
	if err != nil {
		panic(err)
	}
	marshal, err := json.Marshal(search)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(marshal))
}

```


至此先启动服务端，再启动客户端，一个简单的grpc的小demo就完成啦。