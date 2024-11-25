



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



### 介绍

go get 命令可以借助代码管理工具通过远程拉取或更新代码包及其依赖包，并自动完成编译和安装。整个过程就像安装一个 App 一样简单。 这个命令可以动态获取远程代码包，目前支持的有 BitBucket、GitHub、Google Code 和 Launchpad。在使用 go get 命令前，需要安装与远程包匹配的代码管理工具，如 Git、SVN、HG 等，参数中需要提供一个包名。



`go get` 主要被设计为修改 `go.mod` 追加依赖之类的，但还存在类似 `go mod tidy` 之类的命令，所以使用频率可能不会很高；

到目前为止，Go 一直使用 `go get` 命令，将我们需要的工具安装到 `$GOPATH/bin` 目录下，但这种方式存在一个很严重的问题。`go get` 由于具备更改 `go.mod` 文件的能力，因此我们 **必须要避免执行 go get 命令时，让它接触到我们的 go.mod 文件** ，否则它会将我们安装的工具作为一个依赖。





### 语法说明

下载导入路径指定的包及其依赖项，然后安装命名包，即执行go install命令。 用法如下：



**go get**

```bash
go get [-d] [-f] [-t] [-u] [-fix] [-insecure] [build flags] [packages]
```

|   参数    | 描述                                                         |
| :-------: | ------------------------------------------------------------ |
|    -d     | 让命令程序只执行下载动作，而不执行安装动作。                 |
|    -f     | 仅在使用-u标记时才有效。该标记会让命令程序忽略掉对已下载代码包的导入路径的检查。如果下载并安装的代码包所属的项目是你从别人那里Fork过来的，那么这样做就尤为重要了。 |
|   -fix    | 让命令程序在下载代码包后先执行修正动作，而后再进行编译和安装。 |
| -insecure | 允许命令程序使用非安全的scheme（如HTTP）去下载指定的代码包。如果你用的代码仓库（如公司内部的Gitlab）没有HTTPS支持，可以添加此标记。请在确定安全的情况下使用它。 |
|    -t     | 让命令程序同时下载并安装指定的代码包中的测试源码文件中依赖的代码包。 |
|    -u     | 让命令利用网络来更新已有代码包及其依赖包。默认情况下，该命令只会从网络上下载本地不存在的代码包，而不会更新已有的代码包。 |
|    -v     | 打印出被构建的代码包的名字                                   |
|    -x     | 打印出用到的命令                                             |



## go install

> go build命令比较相似，go build命令会编译包及其依赖，生成的文件存放在当前目录下。而且go build只对main包有效，其他包不起作用。而go install对于非main包会生成静态文件放在$GOPATH/pkg目录下，文件扩展名为a。 如果为main包，则会在$GOPATH/bin下生成一个和给定包名相同的可执行二进制文件。具体语法如下:

```bash
go install [-i] [build flags] [packages]
```



## go build



### 使用

```go
go build [-o 输出名] [-i] [编译标记] [包名]
```

- 如果参数为`XX.go`文件或文件列表，则编译为一个个单独的包。
- 当编译单个`main`包（文件），则生成可执行文件。
- 当编译单个或多个包非主包时，只构建编译包，但丢弃生成的对象（`.a`），仅用作检查包可以构建。
- 当编译包时，会自动忽略`_test.go`的测试文件。



### 简单说明

> `go build`的使用比较简洁,所有的参数都可以忽略,直到只有`go build`,这个时候意味着使用当前目录进行编译，下面的几条命令是等价的. 都是使用当前目录编译的意思。因为我们忽略了`packages`,所以自然就使用当前目录进行编译了。

```go
go build
go build .
go build hello.go
```

> 从这里我们也可以推测出,`go build`本质上需要的是一个路径,让编译器可以找到哪些需要编译的`go`文件。 `packages`其实是一个相对路径,是相对于我们定义的`GOROOT`和`GOPATH`这两个环境变量的,所以有了`packages`这个参数后, `go build`就可以知道哪些需要编译的`go`文件了。





## go mod tidy

`go mod tidy` 是 Go 语言的一项工具，用于管理和清理 `go.mod` 文件以及相关的依赖项。它的主要作用是移除未使用的依赖项，并下载缺失的依赖项以确保项目的模块依赖配置是最新且最简洁的。以下是其工作原理的详细解释：



### 1. 更新依赖信息

`go mod tidy` 会检查 `go.mod` 文件和代码中的所有包导入路径，以确保所有依赖项都在 `go.mod` 文件中正确列出。如果某个依赖项在代码中使用了但未在 `go.mod` 文件中声明，它会自动添加进去。



### 2. 移除未使用的依赖项

它会扫描整个项目的代码，找出那些不再被使用的依赖项，并将它们从 `go.mod` 文件中移除。这有助于减少项目的复杂性和构建时间。



### 3. 下载缺失的模块

对于 `go.mod` 文件中声明的依赖项，如果本地模块缓存中缺少这些依赖项，它会自动下载并缓存这些模块。这样可以确保构建环境是完整和一致的。



### 4. 更新 `go.sum` 文件

在执行完上述操作后，`go mod tidy` 会更新 `go.sum` 文件。这个文件包含了所有依赖项的哈希值和版本信息，用于确保每次构建时依赖项的一致性。如果有任何新添加或删除的依赖项，`go.sum` 文件也会相应地更新。







## go.mod 



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