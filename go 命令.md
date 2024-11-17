



## go mod

golang 提供了 `go mod`命令来管理包。

go mod 有以下命令：

| 命令     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| download | download modules to local cache(下载依赖包)                  |
| edit     | edit go.mod from tools or scripts（编辑go.mod                |
| graph    | print module requirement graph (打印模块依赖图)              |
| init     | initialize new module in current directory（在当前目录初始化mod） |
| tidy     | add missing and remove unused modules(拉取缺少的模块，移除不用的模块) |
| vendor   | make vendored copy of dependencies(将依赖复制到vendor下)     |
| verify   | verify dependencies have expected content (验证依赖是否正确） |
| why      | explain why packages or modules are needed(解释为什么需要依赖) |

### go mod tidy

1. 引用项目需要的依赖增加到go.mod文件。
2. 去掉go.mod文件中项目不需要的依赖。

### go mod init

例如 ： go mod init imooc.com.demo



## 导入本地的依赖包

**replace**

例如:

```go
replace imooc.com.demo1 => ../gomod-demo1
```



## 如何管理内部私有包

用于沉淀一些公共包给企业的不同项目使用

避免重复造轮，统一使用规范

GOPRIVATE = *私有仓库域名，不走go proxy

过程：略





## go module 的包校验

为了防止 go module中的包被篡改

go.sum文件保存了依赖包的hash值

GOPRIVATE包将不会做checksum校验

## go get

`go get` 主要被设计为修改 `go.mod` 追加依赖之类的，但还存在类似 `go mod tidy` 之类的命令，所以使用频率可能不会很高；

到目前为止，Go 一直使用 `go get` 命令，将我们需要的工具安装到 `$GOPATH/bin` 目录下，但这种方式存在一个很严重的问题。`go get` 由于具备更改 `go.mod` 文件的能力，因此我们 **必须要避免执行 go get 命令时，让它接触到我们的 go.mod 文件** ，否则它会将我们安装的工具作为一个依赖。



## go.mod 

module:定义当前项目的模块路径

go: 标识当前模块的Go语言版本

require: 说明 Module 需要什么版本的依赖

replace:  源码import 时会被后者代替

exclude 语句可以排除指定依赖包



### 如何编辑

在 Go 1.16 中，另一个行为变更是 `go build` 和 `go test` 不会自动编辑 `go.mod` 了，基于以上信息，Go 1.16 中将进行如下处理：

- 通过在代码中修改 import 语句，来修改 `go.mod`：
  - `go get` 可用于添加新模块；
  - `go mod tidy` 删除掉无用的模块；
- 将未导入的模块写入 `go.mod`:
  - `go get <package>[@<version>]`;
  - `go mod tidy` 也可以；
  - 手动编辑；

由于 `go build` 和 `go test` 不会自动编辑 `go.mod` 了，所以可以将原本的行为通过 `go mod tidy` 共同处理。



## 缓存

缓存：Go Modules会缓存已下载的依赖项。在某些情况下，Go Modules可能会使用缓存的版本而不是最新的版本。您可以尝试清除Go Modules的缓存，然后再运行`go mod tidy`：

```shell
go clean -modcache
go mod tidy
```