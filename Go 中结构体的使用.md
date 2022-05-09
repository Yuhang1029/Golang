# Go 中结构体的使用

参考文档

[Go 语言基础之结构体](https://www.liwenzhou.com/posts/Go/10_struct/)

[Go 继承，多重继承（struct） - Go语言中文网 ](https://studygolang.com/articles/12175)

[Go 中使用继承](https://segmentfault.com/a/1190000022429780)

[Go 中结构体转 JSON 技巧](https://www.liwenzhou.com/posts/Go/json_tricks_in_go/)

---

## 类型别名和自定义类型

自定义类型是定义了一个全新的类型。我们可以基于内置的基本类型定义，也可以通过`struct` 定义。例如：

```go
// 将 MyInt 定义为 int 类型
type MyInt int
```

通过 `type` 关键词定义，`MyInt` 就是一个新的类型，具有 `int` 的性质。这个特性在 `Context` 的使用中，如果需要 `Context` 传值就使用过。

类型别名是`Go1.9`版本添加的新功能。类型别名规定：TypeAlias 只是 Type 的别名，本质上 TypeAlias 与 Type 是同一个类型。

```go
// 类型定义
type NewInt int

// 类型别名
type MyInt = int

func main() {
    var a NewInt
    var b MyInt

    fmt.Printf("type of a:%T\n", a) //type of a:main.NewInt
    fmt.Printf("type of b:%T\n", b) //type of b:int
}
```

结果显示 a 的类型是`main.NewInt`，表示 `main` 包下定义的`NewInt`类型。b 的类型是`int`。`MyInt`类型只会在代码中存在，编译完成时并不会有`MyInt`类型。

## 结构体

Go语言中没有“类”的概念，也不支持“类”的继承等面向对象的概念。Go 语言中通过结构体的内嵌再配合接口，比面向对象具有更高的扩展性和灵活性。

使用 `type` 和 `struct` 关键词来定义一个结构体。只有当结构体实例化时，才开始真正的分配内存，也就是必须实例化才可以使用字段。Go 里面直接通过 `.` 来访问成员变量，**无论是实例还是结构体指针**。

```go
type 类型名 struct {
    name Type
}
```

几种实例化方法汇总：

```go
type person struct {
    name string
    city string
    age  int8
}

func test() {
    var p1 person
    p1.name = "a"
    p1.city = "b"
    p1.age = 18
    fmt.Printf("p1=%#v\n", p1) //p1=main.person{name:"a", city:"b", age:18}

    var p2 = new(person)
    p2.name = "a1"
    p2.city = "b1"
    p2.age = 28
    fmt.Printf("p2=%#v\n", p2) //p2=&main.person{name:"a1", city:"b1", age:28}

    // 也相当于创建了结构体指针
    p3 := &person{}
    ...
}
```

没有初始化的结构体，其成员变量都是对应其类型的零值。初始化可以在实例化的时候直接利用健值对进行赋值，也可以通过构造函数。Go 中没有构造函数，一般是靠自己写一个方法，返回对应结构体的指针。

```go
func newPerson(name, city string, age int8) *person {
    return &person{
        name: name,
        city: city,
        age:  age,
    }
}
```

### 方法和接收者

Go 语言中的方法 (method) 是一种作用于特定类型变量的函数，这种特定的类型变量叫做 接收者 (Receiver)，类似其他语言的 `self` 或者 `this`。接收者可以是值类型或者指针类型，一般都采用指针类型，其修改操作会对本身有效果，且拷贝代价小，即使实际接收者是值类型也可以使用。

```go
func (接收者变量 接收者类型) 方法名(参数列表) (返回参数) {
    函数体
}
```

其中，接收者变量的命名，官方建议是小写字母缩写，例如 Person 可以是 p。

### 结构体的嵌套

一个结构体中可以嵌套另一个结构体或者是对应的结构体指针，Go 通过这种方式实现类似其他语言继承的效果。**结构体中字段大写开头表示可公开访问，小写开头表示私有（只能在定义当前结构体的包中访问）**。下面的代码有详细介绍：

```go
// 继承
// 一个结构体嵌到另一个结构体，称作组合
// 匿名和组合的区别
// 如果一个struct嵌套了另一个匿名结构体，那么这个结构可以直接访问匿名结构体的方法，从而实现继承
// 如果一个struct嵌套了另一个【有名】的结构体，那么这个模式叫做组合
// 如果一个struct嵌套了多个匿名结构体，那么这个结构可以直接访问多个匿名结构体的方法，从而实现多重继承

type ShapeInterface interface {
    GetName() string
}

type Shape struct {
    name string
}

func (s *Shape) GetName() string {
    return s.name
}

type Rectangle struct {
    Shape
    w, h float64
}
```

从上面的例子看出，外层结构体类型通过匿名嵌套一个已命名的结构体类型后就可以获得匿名成员类型的所有导出成员，而且也获得了该类型导出的全部的方法。

## 结构体和 JSON 转换的一些技巧

在 Go 语言中，通过 `json.Marshal()` 和 `json.Unmarshal()` 来实现序列化和反序列化的操作。在结构体中，可以利用 `Tag` 指定一些相应信息。`Tag` 是结构体的源信息，可以在运行时候通过反射机制读取出来。`Tag`在结构体字段的后方定义，由一对**反引号**包裹起来，具体的格式如下：

```go
type Person struct {
    Name   string  `json:"name"`
    Age    int8    `json:"-"`
    Sex    string  `json:"sex", omitempty`
    weight int8
}
```

如上所示，序列化与反序列化默认情况下使用结构体的字段名，我们可以通过给结构体字段添加tag来指定 JSON 序列化生成的字段名。

下面是几个使用的小技巧：

1. 如果你想在 JSON 序列化/反序列化的时候忽略掉结构体中的某个字段，可以按如下方式在tag中添加`-`。同样，如果该字段开头字母是小写的，代表他是私有字段，同样不会序列化出来。

2. 当 struct 中的字段没有值时， `json.Marshal()` 序列化的时候不会忽略这些字段，而是默认输出字段的类型零值（例如`int`和`float`类型零值是 0，`string`类型零值是`""`，对象类型零值是 nil）。如果想要在序列序列化时忽略这些没有值的字段时，可以在对应字段添加`omitempty` tag。

3. 对于结构体的嵌套，如果想要在嵌套的结构体为空时忽略该字段，仅添加`omitempty` 是不够的，还需要把相应的结构体字段从值类型编程指针类型。

4. 有时候，前端在传递来的json数据中可能会使用字符串类型的数字，这个时候可以在结构体 `tag` 中添加`string`来告诉 `json` 包从字符串中解析相应字段的数据：
   
   ```go
   type Card struct {
       ID    int64   `json:"id,string"`    // 添加string tag
       Score float64 `json:"score,string"` // 添加string tag
   }
   
   func intAndStringDemo() {
       jsonStr1 := `{"id": "1234567","score": "88.50"}`
       var c1 Card
       if err := json.Unmarshal([]byte(jsonStr1), &c1); err != nil {
           fmt.Printf("json.Unmarsha jsonStr1 failed, err:%v\n", err)
           return
       }
       fmt.Printf("c1:%#v\n", c1) // c1:main.Card{ID:1234567, Score:88.5}
   }
   ```

        
