## go module使用及本地包导入

## 为什么需要包管理工具？

在go语言早期，所有的第三方库全部存放于GOPATH目录下，这样会导致第三方库的版本错乱。同一个项目，在不同的环境下，可能会因为三方库的版本不同而无法运行，该如何解决？

## go module

go语言的发展过程中，出现了vendor模式，如果项目的目录下有vendor文件，编译器会优先使用vendor文件内的包进行编译。而go module 是在Go1.11版本后，由官方推出的版本管理工具，并且在1.13版本后，go module 成为了Go语言默认的包管理工具。因此着重讲解go module。

### 环境变量设置

打开go module需要设置环境变量GO111MODULE， 该变量默认值为auto。

- GO111MODULE=auto，项目在GOPATH/src以外并且项目根目录中包含go.mod，会使用go module。否则会从vendor和GOPATH中寻找依赖的包。

- GO111MODULE=on，启用go module，只在go.mod中寻找一来的包。

- GO111MODULE=off，禁用go module，只在vendor和GOPATH中寻找依赖的包。

GOPROXY简单来说就是一个代理，让我们更方便的下载某些由于墙的原因而导致无法下载的第三方包。

如果你使用的的是Go1.13及以上

```go
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.io,direct
```

如果你使用的的是Go1.12及以下

```go
//windows
set GO111MODULE=on
set GOPROXY=https://goproxy.io
//linux
export GO111MODULE=on
export GOPROXY=https://goproxy.io
```

### go mod常用命令

可通过go help mod 获取

```go
go mod download    将依赖的模块到本地缓存（默认为$GOPATH/pkg/mod目录）
go mod edit        通过工具或脚本编辑go.mod文件
go mod graph       打印模块依赖图
go mod init        初始化当前文件夹, 创建go.mod文件
go mod tidy        增加缺少的模块，删除无用的模块
go mod vendor      将依赖复制到vendor下
go mod verify      依赖校验
go mod why         解释为什么需要包或依赖
```

### go.mod文件

go.mod文件记录该项目所有的三方包信息，如下：
$$

$$

```go
module go_project/Futures-Go-demo

go 1.14

require (
	github.com/dgrijalva/jwt-go v3.2.0+incompatible
	github.com/gorilla/websocket v1.4.0
	github.com/spf13/cast v1.3.0
	golang.org/x/net latest
)
```

#### 三方包的版本

go mod支持语义化版本号，比如`go get foo@v1.2.3`，也可以跟git的分支或tag，比如`go get foo@master`，当然也可以跟git提交哈希，比如`go get foo@e3702bed2`。关于依赖的版本支持以下几种格式：

```go
gopkg.in/tomb.v1 v1.0.0-20141024135613-dd632973f1e7
gopkg.in/vmihailenco/msgpack.v2 v2.9.1
gopkg.in/yaml.v2 <=v2.2.1
github.com/tatsushid/go-fastping v0.0.0-20160109021039-d7bb493dee3e
latest
```

#### replace设置

go.mod中使用replace替换成github上对应的库，如：

```
replace (
	golang.org/x/net v0.0.0-20180821023952-922f4815f713 => github.com/golang/net v0.0.0-20180826012351-8a410e7b638d
)
```

## go module导入本地包

go module导入本地包是初学者在初次使用go module时常常会踩的坑，下面分两种情况讲解。

### 同一个项目下

#### 目录结构

```
test
├── go.mod
├── main
│	└── main.go
└── mypackage
    └── mypackage.go
```

#### 导入

go.mod文件下：

```go
module test

go 1.14
```

test/main/main.go文件下“

```go
package main

import (
	"fmt"
	"test/mypackage"
)
```

## 不在同一个项目下

当导入包不在同一个项目下时，需要使用replace的方法。

#### 目录结构

```
├── test
│   ├── main.go
│   └── go.mod
└── newpackage
    ├── newpackage.go
    └── go.mod

```

#### 导入

由于不在一个项目内，需要使用replace。

```go
module test

go 1.14

require "newpackage" v0.0.0
replace "newpackage" => "../newpackage"
```

包调用同上，不再赘述。