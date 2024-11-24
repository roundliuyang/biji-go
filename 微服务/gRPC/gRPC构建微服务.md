# gRPC构建微服务



代码实现查询，客户端向服务端查询用户的信息

## 编写proto文件

```go
// 版本号
syntax = "proto3";

// 指定包名
option go_package = "./proto;proto";

// 定义结构体
message UserRequest {
  // 定义用户名
  string name = 1;
}

// 响应结构体
message UserResponse {
  int32 id = 1;
  string name = 2;
  int32 age = 3;
  // repeated修饰符是可变数组，go转切片
  repeated string hobby = 4;
}

// service定义方法
service UserInfoService {
  rpc GetUserInfo (UserRequest) returns (UserResponse) {
  }
}
```



## 生成.go文件

```
goland中打开命令行，输入命令生成接口文件：protoc -I . --go_out=. --go-grpc_out=. ./user.proto
```





## 编写服务端

```go
package main

// 1.需要监听
// 2.需要实例化gRPC服务端
// 3.在gRPC商注册微服务
// 4.启动服务端
import (
   "context"
   "fmt"
   pb "/gRPC/proto"  //注意这个路径
   "google.golang.org/grpc"
   "net"
)

// 定义空接口
type UserInfoService struct{}

var u = UserInfoService{}

// 实现方法
func (s *UserInfoService) GetUserInfo(ctx context.Context, req *pb.UserRequest) (resp *pb.UserResponse, err error) {
   // 通过用户名查询用户信息
   name := req.Name
   // 数据里查用户信息
   if name == "zs" {
      resp = &pb.UserResponse{
         Id:    1,
         Name:  name,
         Age:   22,
         Hobby: []string{"Sing", "Run"},
      }
   }
   return
}

func main() {
   // 地址
   addr := "127.0.0.1:8080"
   // 1.监听
   listener, err := net.Listen("tcp", addr)
   if err != nil {
      fmt.Printf("监听异常:%s\n", err)
   }
   fmt.Printf("监听端口：%s\n", addr)
   // 2.实例化gRPC
   s := grpc.NewServer()
   // 3.在gRPC上注册微服务
   pb.RegisterUserInfoServiceServer(s, &u)
   // 4.启动服务端
   s.Serve(listener)
}
```



## 编写客户端

```go
package main

import (
   "context"
   "fmt"
   pb "/gRPC/proto"   //注意这个路径
   "google.golang.org/grpc"
)

// 1.连接服务端
// 2.实例gRPC客户端
// 3.调用

func main() {
   // 1.连接
   conn, err := grpc.Dial("127.0.0.1:8080", grpc.WithInsecure())
   if err != nil {
      fmt.Printf("连接异常： %s\n", err)
   }
   defer conn.Close()
   // 2. 实例化gRPC客户端
   client := pb.NewUserInfoServiceClient(conn)
   // 3.组装请求参数
   req := new(pb.UserRequest)
   req.Name = "zs"
   // 4. 调用接口
   response, err := client.GetUserInfo(context.Background(), req)
   if err != nil {
      fmt.Println("响应异常  %s\n", err)
   }
   fmt.Printf("响应结果： %v\n", response)
}
```