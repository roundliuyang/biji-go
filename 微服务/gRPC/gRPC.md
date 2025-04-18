# gRPC



## gRPC简介

- gRPC由google开发，是一款语言中立、平台中立、开源的远程过程调用系统
- gRPC客户端和服务端可以在多种环境中运行和交互，例如用java写一个服务端，可以用go语言写客户端调用



## gRPC与Protobuf介绍

- 微服务架构中，由于每个服务对应的代码库是独立运行的，无法直接调用，彼此间的通信就是个大问题
- gRPC可以实现微服务，将大的项目拆分为多个小且独立的业务模块，也就是服务，各服务间使用高效的protobuf协议进行RPC调用，gRPC默认使用protocol buffers，这是google开源的一套成熟的结构数据序列化机制（当然也可以使用其他数据格式如JSON）
- 可以用proto files创建gRPC服务，用message类型来定义方法参数和返回类型





## 安装gRPC和Protobuf

- go get github.com/golang/protobuf/proto
- go get google.golang.org/grpc（无法使用，用如下命令代替）
  - git clone https://github.com/grpc/grpc-go.git $GOPATH/src/google.golang.org/grpc
  - git clone https://github.com/golang/net.git $GOPATH/src/golang.org/x/net
  - git clone https://github.com/golang/text.git $GOPATH/src/golang.org/x/text
  - go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
  - git clone https://github.com/google/go-genproto.git $GOPATH/src/google.golang.org/genproto
  - cd $GOPATH/src/
  - go install google.golang.org/grpc
- go get github.com/golang/protobuf/protoc-gen-go
- 上面安装好后，会在GOPATH/bin下生成protoc-gen-go.exe
- 但还需要一个protoc.exe，windows平台编译受限，很难自己手动编译，直接去网站下载一个，地址：https://github.com/protocolbuffers/protobuf/releases/tag/v3.9.0 ，同样放在GOPATH/bin下

注意：这里面好多都是需要vpn才能下载好的！分享一个下载好的https://pan.baidu.com/s/1T8eJkHib2uPL3gNMCRdZmQ 提取码 s42z