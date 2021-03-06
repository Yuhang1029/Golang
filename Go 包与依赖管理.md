# Go 中包与依赖管理

参考文档

[Go 语言基础之包](https://www.liwenzhou.com/posts/Go/11-package/)

## 包 (package)

### 定义

在 Go 语言中使用包来支持代码的模块化和代码复用，一个包是由一个或多个 Go 源码文件组成的（即 `.go` 结尾的文件），Go 语言本身提供了许多内置包，例如 `fmt`，`os` 等。 

我们可以根据自己的需要创建自定义的包，一个包可以简单理解成一个存放 `.go` 的文件的文件夹，该文件夹下面所有`.go` 的文件的第一行都需要有申明。一般情况下包名和文件夹名保持一致。

```go
package packageName
```

需要注意一个文件夹下面直接包含的文件只能归属一个包，同一个包的文件不能在多个文件夹下。包名为`main`的包是应用程序的入口包，这种包编译后会得到一个可执行文件，而编译不包含`main`包的源代码则不会得到可执行文件。

### 包的可见性

在同一个包内部声明的标识符都位于同一个命名空间下，在不同的包内部声明的标识符就属于不同的命名空间。想要在包的外部使用包内部的标识符就需要添加包名前缀，例如`fmt.Println("Hello world!")`，就是指调用`fmt`包中的`Println`函数。

如果想让一个包中的标识符（如变量、常量、类型、函数等）能被外部的包使用，那么标识符必须是对外可见的（public）。在Go语言中是通过标识符的首字母大/小写来控制标识符的对外可见（public）/不可见（private）的。**在一个包内部只有首字母大写的标识符才是对外可见的**。

### 包的引入

要在当前包中使用另外一个包的内容就需要使用`import`关键字引入这个包，并且 `import` 语句通常放在文件的开头，`package`声明语句的下方。完整的引入声明语句格式如下:

```go
import importPackageName "path/to/package"
```

其中：

- importPackageName：引入的包名，**通常都省略，默认值为引入包的包名。**

- path/to/package：引入包的路径名称，必须使用双引号包裹起来。

- **Go语言中禁止循环导入包**。

一个 Go 源码文件中可以同时引入多个包，例如：

```go
import "fmt"
import "net/http"
import "os"
```

当然可以使用批量引入的方式，这是默认使用的方式。

```go
import (
    "fmt"
    "net/http"
    "os"
)
```

当引入的多个包中存在相同的包名或者想自行为某个引入的包设置一个新包名时，都需要通过`importPackageName`指定一个在当前文件中使用的新包名。

如果引入一个包的时候为其设置了一个特殊`_`作为包名，那么这个包的引入方式就称为匿名引入。一个包被匿名引入的目的主要是为了加载这个包，从而使得这个包中的资源得以初始化。 被匿名引入的包中的`init`函数将被执行并且仅执行一遍。

```go
import _ "github.com/go-sql-driver/mysql"
```

匿名引入的包与其他方式导入的包一样都会被编译到可执行文件中。**需要注意的是，Go语言中不允许引入包却不在代码中使用这个包的内容**，如果引入了未使用的包则会触发编译错误。

### init() 初始化函数

在每一个Go源文件中，都可以定义任意个如下格式的特殊函数：

```go
func init(){
  // ...
}
```

这种特殊的函数不接收任何参数也没有任何返回值，我们也不能在代码中主动调用它。当程序启动的时候，`init()` 函数会按照它们声明的顺序自动执行。一个包的初始化过程是按照代码中引入的顺序来进行的，所有在该包中声明的 `init()` 函数都将被串行调用并且仅调用执行一次。每一个包初始化的时候都是先执行依赖的包中声明的 `init()` 函数再执行当前包中声明的 `init()` 函数。确保在程序的 `main` 函数开始执行时所有的依赖包都已初始化完成。

## 依赖管理

在 Go 语言的早期版本中，我们编写 Go 项目代码时所依赖的所有第三方包都需要保存在GOPATH 这个目录下面。这样的依赖管理方式存在一个致命的缺陷，那就是不支持版本管理，同一个依赖包只能存在一个版本的代码，可是我们本地的多个项目完全可能分别依赖同一个第三方包的不同版本。

Go module 是 Go1.11 版本发布的依赖管理方案，从 Go1.14 版本开始推荐在生产环境使用，于 Go1.16 版本默认开启。Go module 提供了以下命令供我们使用：

| 命令              | 介绍                        |
| --------------- | ------------------------- |
| go mod init     | 初始化项目依赖，生成 go.mod 文件      |
| go mod download | 根据 go.mod 文件下载依赖          |
| go mod tidy     | 比对项目文件中引入的依赖与 go.mod 进行比对 |
| go mod verify   | 检验一个依赖包是否被篡改过             |
