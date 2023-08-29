---
title: GRPC
date: 2023-07-03 16:41:38
tags:
- RPC
- golang
mathjax: true
categories: 
- RPC
- golang
---

### GRPC概述

> 服务和服务之间调用需要使用RPC，`gRPC`是一款**语言中立**、**平台中立**、开源的**远程过程调用系统**，`gRPC`客户端和服务端可以在多种环境中运行和交互，例如用`java`写一个服务端，可以用`go`语言写客户端调用

数据在进行网络传输的时候，需要进行序列化，序列化协议有很多种，比如`xml`,` json`，`protobuf`等

gRPC默认使用`protocol buffers`，这是google开源的一套成熟的结构数据序列化机制。

在学习gRPC之前，需要先了解`protocol buffers`

**序列化**：将数据结构或对象转换成二进制串的过程。

**反序列化**：将在序列化过程中所产生的二进制串转换成数据结构或对象的过程。

### protobuf

protobuf是谷歌开源的一种数据格式，适合高性能，对响应速度有要求的数据传输场景。因为profobuf是二进制数据格式，需要编码和解码。数据本身不具有可读性。因此只能反序列化之后得到真正可读的数据。

优势：

1. 序列化后体积相比Json和XML很小，适合网络传输
2. 支持跨平台多语言
3. 消息格式升级和兼容性还不错
4. 序列化反序列化速度很快

####  安装

- 第一步：下载通用编译器

  地址：https://github.com/protocolbuffers/protobuf/releases

  根据不同的操作系统，下载不同的包，我是windows电脑，解压出来是`protoc.exe`

  ![image-20220423001259067](https://www.mszlu.com/assets/image-20220423001259067.b9d637b9.png)

- 第二步：配置环境变量

  ![image-20220423002031614](data:image/png;base64,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)

- 第三步：安装go专用的protoc的生成器

```go
go get github.com/golang/protobuf/protoc-gen-go
```

1

安装后会在`GOPATH`目录下生成可执行文件，protobuf的编译器插件`protoc-gen-go`，执行`protoc`命令会自动调用这个插件

> 如何使用protobuf呢？

1. 定义了一种源文件，扩展名为 `.proto`，使用这种源文件，可以定义存储类的内容(消息类型)
2. protobuf有自己的编译器 `protoc`，可以将 `.proto` 编译成对应语言的文件，就可以进行使用了

####  hello world

> 假设，我们现在需要传输用户信息，其中有username和age两个字段

```protobuf
// 指定的当前proto语法的版本，有2和3
syntax = "proto3";
//option go_package = "path;name"; ath 表示生成的go文件的存放地址，会自动生成目录的
// name 表示生成的go文件所属的包名
option go_package="../service";
// 指定等会文件生成出来的package
package service;

message User {
  string username = 1;
  int32 age = 2;
}
```

1
2
3
4
5
6
7
8
9
10
11
12
13

**运行protoc命令编译成go中间文件**

```go
# 编译user.proto之后输出到service文件夹
protoc --go_out=../service user.proto
```

1
2

**测试**

```go
package main

import (
	"fmt"
	"google.golang.org/protobuf/proto"
	"testProto/service"
)

func main()  {
	user := &service.User{
		Username: "mszlu",
		Age: 20,
	}
	//转换为protobuf
	marshal, err := proto.Marshal(user)
	if err != nil {
		panic(err)
	}
	newUser := &service.User{}
	err = proto.Unmarshal(marshal, newUser)
	if err != nil {
		panic(err)
	}
	fmt.Println(newUser.String())
}
```

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26

####  proto文件介绍

#####  message介绍

`message`：`protobuf`中定义一个消息类型是通过关键字`message`字段指定的。

消息就是需要传输的数据格式的定义。

message关键字类似于C++中的class，Java中的class，go中的struct

例如：

```protobuf
message User {
  string username = 1;
  int32 age = 2;
}
```

在消息中承载的数据分别对应于每一个字段。

其中每个字段都有一个名字和一种类型 。

#####  字段规则

- `required`:消息体中必填字段，不设置会导致编解码异常。（例如位置1）
- `optional`: 消息体中可选字段。（例如位置2）
- `repeated`: 消息体中可重复字段，重复的值的顺序会被保留（例如位置3）在go中重复的会被定义为切片。

```protobuf
message User {
  string username = 1;
  int32 age = 2;
  optional string password = 3;
  repeated string address = 4;
}
```

#####   字段映射

| **.proto Type** | **Notes**                                                    | **C++ Type** | **Python Type** | **Go Type** |
| --------------- | ------------------------------------------------------------ | ------------ | --------------- | ----------- |
| double          |                                                              | double       | float           | float64     |
| float           |                                                              | float        | float           | float32     |
| int32           | 使用变长编码，对于负值的效率很低，如果你的域有 可能有负值，请使用sint64替代 | int32        | int             | int32       |
| uint32          | 使用变长编码                                                 | uint32       | int/long        | uint32      |
| uint64          | 使用变长编码                                                 | uint64       | int/long        | uint64      |
| sint32          | 使用变长编码，这些编码在负值时比int32高效的多                | int32        | int             | int32       |
| sint64          | 使用变长编码，有符号的整型值。编码时比通常的 int64高效。     | int64        | int/long        | int64       |
| fixed32         | 总是4个字节，如果数值总是比总是比228大的话，这 个类型会比uint32高效。 | uint32       | int             | uint32      |
| fixed64         | 总是8个字节，如果数值总是比总是比256大的话，这 个类型会比uint64高效。 | uint64       | int/long        | uint64      |
| sfixed32        | 总是4个字节                                                  | int32        | int             | int32       |
| sfixed32        | 总是4个字节                                                  | int32        | int             | int32       |
| sfixed64        | 总是8个字节                                                  | int64        | int/long        | int64       |
| bool            |                                                              | bool         | bool            | bool        |
| string          | 一个字符串必须是UTF-8编码或者7-bit ASCII编码的文 本。        | string       | str/unicode     | string      |
| bytes           | 可能包含任意顺序的字节数据。                                 | string       | str             | []byte      |

#####  默认值

protobuf3 删除了 protobuf2 中用来设置默认值的 default 关键字，取而代之的是protobuf3为各类型定义的默认值，也就是约定的默认值，如下表所示：

| 类型     | 默认值                                                       |
| :------- | :----------------------------------------------------------- |
| bool     | false                                                        |
| 整型     | 0                                                            |
| string   | 空字符串""                                                   |
| 枚举enum | 第一个枚举元素的值，因为Protobuf3强制要求第一个枚举元素的值必须是0，所以枚举的默认值就是0； |
| message  | 不是null，而是DEFAULT_INSTANCE                               |

#####  标识号

`标识号`：在消息体的定义中，每个字段都必须要有一个唯一的标识号，标识号是[0,2^29-1]范围内的一个整数。

```protobuf
message Person { 

  string name = 1;  // (位置1)
  int32 id = 2;  
  optional string email = 3;  
  repeated string phones = 4; // (位置4)
}
```

以Person为例，name=1，id=2, email=3, phones=4 中的1-4就是标识号。

#####  定义多个消息类型

一个proto文件中可以定义多个消息类型

```go
message UserRequest {
  string username = 1;
  int32 age = 2;
  optional string password = 3;
  repeated string address = 4;
}

message UserResponse {
  string username = 1;
  int32 age = 2;
  optional string password = 3;
  repeated string address = 4;
}
```

#### 嵌套消息

可以在其他消息类型中定义、使用消息类型，在下面的例子中，Person消息就定义在PersonInfo消息内，如 ：

```protobuf
message PersonInfo {
    message Person {
        string name = 1;
        int32 height = 2;
        repeated int32 weight = 3;
    } 
	repeated Person info = 1;
}
```

如果你想在它的父消息类型的外部重用这个消息类型，你需要以PersonInfo.Person的形式使用它，如：

```protobuf
message PersonMessage {
	PersonInfo.Person info = 1;
}
```

当然，你也可以将消息嵌套任意多层，如 :

```protobuf
message Grandpa { // Level 0
    message Father { // Level 1
        message son { // Level 2
            string name = 1;
            int32 age = 2;
    	}
	} 
    message Uncle { // Level 1
        message Son { // Level 2
            string name = 1;
            int32 age = 2;
        }
    }
}
```

#### 定义服务(Service)

如果想要将消息类型用在RPC系统中，可以在.proto文件中定义一个RPC服务接口，protocol buffer 编译器将会根据所选择的不同语言生成服务接口代码及存根。

```protobuf
service SearchService {
	//rpc 服务的函数名 （传入参数）返回（返回参数）
	rpc Search (SearchRequest) returns (SearchResponse);
}
```

上述代表表示，定义了一个RPC服务，该方法接收SearchRequest返回SearchResponse

###  gRPC实例

####  服务端

```go
// 这个就是protobuf的中间文件

// 指定的当前proto语法的版本，有2和3
syntax = "proto3";
option go_package="../service";

// 指定等会文件生成出来的package
package service;

// 定义request model
message ProductRequest{
	int32 prod_id = 1; // 1代表顺序
}

// 定义response model
message ProductResponse{
	int32 prod_stock = 1; // 1代表顺序
}

// 定义服务主体
service ProdService{
    // 定义方法
    rpc GetProductStock(ProductRequest) returns(ProductResponse);
}
```

生成：

```bash
protoc --go_out=plugins=grpc:./ .\product.proto
```

服务端：

```go
import "google.golang.org/grpc"

func main()  {
	server := grpc.NewServer()
	service.RegisterProdServiceServer(server,service.ProductService)

	listener, err := net.Listen("tcp", ":8002")
	if err != nil {
		log.Fatal("服务监听端口失败", err)
	}
	_ = server.Serve(listener)
}
```

####  客户端

新建client目录，把上述生成的product.pb.go copy过来

```go
func main()  {
	// 1. 新建连接，端口是服务端开放的8002端口
	// 没有证书会报错
	conn, err := grpc.Dial(":8002", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatal(err)
	}

	// 退出时关闭链接
	defer conn.Close()

	// 2. 调用Product.pb.go中的NewProdServiceClient方法
	productServiceClient := service.NewProdServiceClient(conn)

	// 3. 直接像调用本地方法一样调用GetProductStock方法
	resp, err := productServiceClient.GetProductStock(context.Background(), &service.ProductRequest{ProdId: 233})
	if err != nil {
		log.Fatal("调用gRPC方法错误: ", err)
	}

	fmt.Println("调用gRPC方法成功，ProdStock = ", resp.ProdStock)
}
```

代码放在这里了。

https://github.com/thewisecirno/GPRC_Test

### 安全认证

#### 生成自签证书

> 生产环境可以购买证书或者使用一些平台发放的免费证书

- 安装openssl

  网站下载：http://slproweb.com/products/Win32OpenSSL.html

  （mac电脑 自行搜索安装）

- 生成私钥文件

  ```bash
  ## 需要输入密码
  openssl genrsa -des3 -out ca.key 2048
  ```

  

- 创建证书请求

  ```bash
  openssl req -new -key ca.key -out ca.csr
  ```

  

- 生成ca.crt

  ```bash
  openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
  ```

  

找到openssl.cnf 文件

1. 打开copy_extensions = copy

2. 打开 req_extensions = v3_req

3. 找到[ v3_req ],添加 subjectAltName = @alt_names

4. 添加新的标签 [ alt_names ] , 和标签字段

   ```ini
   [ alt_names ]
   
   DNS.1 = *.mszlu.com
   ```

   生成证书私钥server.key

   ```bash
   openssl genpkey -algorithm RSA -out server.key
   ```

5. 通过私钥server.key生成证书请求文件server.csr

   ```bash
   openssl req -new -nodes -key server.key -out server.csr -days 3650 -config ./openssl.cnf -extensions v3_req
   ```

6. 生成SAN证书

   ```bash
   openssl x509 -req -days 365 -in server.csr -out server.pem -CA ca.crt -CAkey ca.key -CAcreateserial -extfile ./openssl.cnf -extensions v3_req
   ```

- **key：** 服务器上的私钥文件，用于对发送给客户端数据的加密，以及对从客户端接收到数据的解密。
- **csr：** 证书签名请求文件，用于提交给证书颁发机构（CA）对证书签名。
- **crt：** 由证书颁发机构（CA）签名后的证书，或者是开发者自签名的证书，包含证书持有人的信息，持有人的公钥，以及签署者的签名等信息。
- **pem：** 是基于Base64编码的证书格式，扩展名包括PEM、CRT和CER。

什么是 SAN？

SAN（Subject Alternative Name）是 SSL 标准 x509 中定义的一个扩展。使用了 SAN 字段的 SSL 证书，可以扩展此证书支持的域名，使得一个证书可以支持多个不同域名的解析。

####  服务端应用证书

将`server.key`和`server.pem` copy到程序中

```go
func main()  {

	//添加证书
	file, err2 := credentials.NewServerTLSFromFile("keys/mszlu.pem", "keys/mszlu.key")
	if err2 != nil {
		log.Fatal("证书生成错误",err2)
	}
	rpcServer := grpc.NewServer(grpc.Creds(file))

	service.RegisterProdServiceServer(rpcServer,service.ProductService)

	listener ,err := net.Listen("tcp",":8002")
	if err != nil {
		log.Fatal("启动监听出错",err)
	}
	err = rpcServer.Serve(listener)
	if err != nil {
		log.Fatal("启动服务出错",err)
	}
	fmt.Println("启动grpc服务端成功")
}
```

#### 客户端认证

公钥copy到客户端

```go
func main()  {
	file, err2 := credentials.NewClientTLSFromFile("client/keys/mszlu.pem", "*.mszlu.com")
	if err2 != nil {
		log.Fatal("证书错误",err2)
	}
	conn, err := grpc.Dial(":8002", grpc.WithTransportCredentials(file))

	if err != nil {
		log.Fatal("服务端出错，连接不上",err)
	}
	defer conn.Close()

	prodClient := service.NewProdServiceClient(conn)

	request := &service.ProductRequest{
		ProdId: 123,
	}
	stockResponse, err := prodClient.GetProductStock(context.Background(), request)
	if err != nil {
		log.Fatal("查询库存出错",err)
	}
	fmt.Println("查询成功",stockResponse.ProdStock)
}
```

#### 单向认证

上述认证方式为单向认证：

![img](https://www.mszlu.com/assets/1586953-20210625171059706-1447106002-16509094111532.8283e9e7.png)

中间人攻击

####  双向认证

![img](https://www.mszlu.com/assets/1586953-20210625211235069-195172761-16509094417774.1dfd9a90.png)

上面的server.pem和server.key 是服务端的 公钥和私钥。

如果双向认证，客户端也需要生成对应的公钥和私钥。