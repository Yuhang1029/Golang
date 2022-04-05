# Go 语言的错误处理

**参考文档** 

[Effective Error Handling in Golang - Earthly Blog](https://earthly.dev/blog/golang-errors/)

[Handle errors in Go (Golang) with errors.Is() and errors.As()](https://gosamples.dev/check-error-type/)

在 Go 里面，错误并不是特殊存在的，他只是一个普通的值。可以定义类型来表示错误，也可以任意创建错误实例，只要实现了它的接口 error

```go
type error interface {
    Error() string
}
```

## 创建错误

发生错误时，可以用 `errors.New` 来创建仅包含简单字符串的错误，或者用 `fmt.Errorf` 创建格式化字符串的错误。例如：

```go
fmt.Errorf("not enough arguments")
errors.New("xxx is wrong")
```

利用字符串包含错误信息有时候并不够，还可以自定义错误结构体，包含错误提示信息以外的其他有用的结构化信息。例如标准库中 `os.Open` 文件操作触发的 `PathError` ，包含操作，文件路径，系统调用等错误信息。

```go
// PathError records an error and the operation and file path that caused it
type PathError struct {
    Op        string
    Path      string
    Err       error
}

func (e PathError) Error() string {
    return e.Op + " "+ e.Path + ": " + e.Err.Error()
}
```

## 传递错误

遇到**当前包内其他函数返回的错误**，一般直接返回。

```go
if err != nil {
    return err
}
```

此外，还可以通过 `errors.WithMessage()` 方法添加该层的上下文信息，确保最上层打印的时候能够拿到最详细的上下文内容。

```go
func foo() error {
    return errors.WithMessage(sql.ErrNoRows, "foo failed")
}
```

**调用其他包，包括标准库返回的错误，包装后再返回**。可以采用 `fmt` ，注意错误值对应的格式符号是 `%w` 。

```go
func Query() error {
    return fmt.Errorf("Query failed: %w", sql.ErrNoRows)
}
```

需要注意，标准库的 `errors` 不记录错误栈，如果需要完整打印，推荐使用 `github.com/pkg/errors` 来代替，格式符是 `%+v` 。该包的 `errors.New` 不仅记录字符串，同时还会保留当前程序计数器的状态。例如下面的例子

```go
package main

import (
    "github.com/pkg/errors"
    "fmt"
)

func A() error {
    return errors.New("NullPointerException")
}

func B() error {
    return A()
}

func main() {
    fmt.Printf("Error: %+v", B())
}
```

对应的输出是

```bash
Error: NullPointerException
main.A
        /Users/yuhangliu/go/src/project1/main/main.go:9
main.B
        /Users/yuhangliu/go/src/project1/main/main.go:13
main.main
        /Users/yuhangliu/go/src/project1/main/main.go:17
runtime.main
        /usr/local/go/src/runtime/proc.go:255
runtime.goexit
        /usr/local/go/src/runtime/asm_amd64.s:1581
Process finished with the exit code 0
```

## 判断错误

最常见的做法是利用类型断言，这也是在 Go 中判断结构体常用的做法。

```go
if peer, ok := err.(*os.PathError); ok {
    fmt.Println(perr.Path)
}
```

除此之外，更推荐使用`errors.Is()` 和  `errors.As()`

```go
var perr *os.PathError

if errors.Is(err, sql.ErrNoRows) {
    ...
}

if errors.As(err, &perr) {
    fmt.Println(perr.Path)
}
```

两者都是对整个错误链上进行校验，即使是错误中途被包裹也会被发现。注意这里两者的区别是 `errors.Is()` 是完全看当前错误是不是某一个特殊结构体，而 `errors.As()` 检查是不是满足某一个特定类型（所以参数是指针）。

## 处理错误

遵循的原则是错误只处理一次，无论是向上抛出还是打日志都是一种处理方法。

## Panic 机制

Go 语言用更轻量的 `panic & recover` 机制取代其他语言的 `try-catch` ，它是用来处理真正的异常的（无法预测的错误而不是普通的错误）。在多层嵌套中调用 `panic` ，可以马上中止当前函数的执行，沿着调用链向上直到最顶端，并且执行每一层的 `defer` 。**注意 `recover` 只在 `defer` 函数中调用才有用。**
