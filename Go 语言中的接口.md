# Go 语言中的接口

参考文档

[Go 语言基础之接口](https://www.liwenzhou.com/posts/Go/12-interface/#autoid-1-2-2)

[如何优雅使用接口](https://zhuanlan.zhihu.com/p/63219494)



### 接口类型

在 Go 语言中接口 (interface) 是一种类型，一种抽象的类型。他定义了一种行为规范，只定义不实现，由具体的结构体来实现规范的细节。

```go
type 接口类型名称 interface {
    方法名称（参数列表）返回值列表
}
```

接口就是规定了一个需要实现的方法列表，**在 Go 语言中一个类型只要实现了接口中规定的所有方法就认为它实现了这个接口，在 Go 中接口的实现不需要显性申明**。



接口类型默认是一个指针（引用类型），如果没有初始化就使用会输出 nil。 





## 接口与类型

一个类型可以同时实现多个接口，而接口间彼此独立，不知道对方的实现。同一类型实现不同的接口不影响使用。同样，不同的类型还可以实现同一个接口，每个结构体可以根据自己的逻辑实现同一个接口。**接口的定义并不规定实现者是否应该用指针接收者还是值接收者来实现接口**。



一个接口的所有方法，不一定需要由一个类型完全实现，接口的方法可以通过在类型中嵌入其他类型或者结构体来实现。

```go
// WashingMachine 洗衣机
type WashingMachine interface {
	wash()
	dry()
}

// 甩干器
type dryer struct{}

// 实现WashingMachine接口的dry()方法
func (d dryer) dry() {
	fmt.Println("甩一甩")
}

// 海尔洗衣机
type haier struct {
	dryer //嵌入甩干器
}

// 实现WashingMachine接口的wash()方法
func (h haier) wash() {
	fmt.Println("洗刷刷")
}
```





## 接口组合

接口与接口之间可以通过互相嵌套形成新的接口类型，例如Go标准库`io`源码中就有很多接口之间互相组合的示例。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

// ReadWriter 是组合Reader接口和Writer接口形成的新接口类型
type ReadWriter interface {
	Reader
	Writer
}
```



接口同样可以作为结构体的一个字段，例如 Go 标准库中 `sort` 的源码：

```go
// Interface 定义通过索引对元素排序的接口类型
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}


// reverse 结构体中嵌入了Interface接口
type reverse struct {
    Interface
}
```

通过在结构体中嵌入一个接口类型，从而让该结构体类型实现了该接口类型，并且还可以改写该接口的方法。

```go
// Less 为reverse类型添加Less方法，重写原Interface接口类型的Less方法
func (r reverse) Less(i, j int) bool {
    return r.Interface.Less(j, i)
}
```

`Interface`类型原本的`Less`方法签名为`Less(i, j int) bool`，此处重写为`r.Interface.Less(j, i)`，即通过将索引参数交换位置实现反转。





## 空接口

空接口是指没有定义任何方法的接口类型。因此任何类型都可以视为实现了空接口。也正是因为空接口类型的这个特性，空接口类型的变量可以存储任意类型的值。通常我们在使用空接口类型时不必使用`type`关键字声明，可以像下面的代码一样直接使用`interface{}`。

```go
var temp interface{}
```

空接口的作用有：

* 使用空接口可以接收任意类型的函数参数

* 可以在 map 中保存任意的值 `map[string]interface{}`





## 类型断言

因为接口可能被赋值为任意类型，想要从接口值中获取到相应类型需要使用类型断言。

```go
v, ok = x.(T)
```

其中：

* x 表示接口类型的变量

* T 表示可能的接口

该语法返回两个参数，第一个是 x 转化成 T 类型后的变量，第二个是一个布尔值，如果是 true 代表断言成功，x 实现了该接口。
