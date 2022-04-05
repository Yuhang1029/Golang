# Go 语言的数据结构

参考文档

[Go 语言基础之数组](https://www.liwenzhou.com/posts/Go/05_array/)

[Go 语言基础之切片](https://www.liwenzhou.com/posts/Go/06_slice/)

[Go 语言基础之哈希表](https://www.liwenzhou.com/posts/Go/08_map/)

[Go 中的 nil 切片 | Origin](https://blog.singee.me/2020/09/24/e8cb67835ea44243b136e3cdf8d5ea84/)

## 数组 (Array)

数组是同一种元素的数据集合。在 Go 语言中，数组从申明的时候就确认，使用时可以修改数组成员，但是大小不可以改变。基本语法：

```go
var 数组变量名 [元素数量]T

// 定义一个长度为3，类型是int的数组a
var a [3]int
```

特别注意的是，数组的长度必须是常量，它是数组类型的一部分，一旦定义不能修改，`[5]int` 和 `[10]int` 是不同的类型。

数组的初始化有以下几种方法：

```go
var a [3]int            // 数组会初始化为int类型的默认值
var b = [3]int{1, 2}    // 使用指定值完成初始化

// 利用编译器自行推断长度
// 这里注意多层数组只能在第一层（最外层）使用...让编译器自行推倒
var c = [...]int{1, 2}
var d = [...][2]string{
    {"a", "a1"},
    {"b", "b1"},
}
```

数组的遍历有两种方法：

```go
func test() {
    a := [...]{1, 2, 3}
    // 方法1
    for i:= 0; i < len(a); i++{
        ...
    }

    // 方法2
    for index, value := range a{
        ...
    }
}
```

需要特别强调的是，在 Go 语言中，数组是**值类型，赋值和传参会复制整个数组**，因此改变副本的值不会对本身有影响。

## 切片 (slice)

切片可以理解是动态数组，是一个拥有相同类型元素的可变长度的序列。它是基于数组类型做的封装，可以自动扩容。**切片是一个引用类型**，内部结构包括指针，长度和容量，可以通过 `len()` 求长度，`cap()` 求容量。

```go
var 数组变量名 []T
```

![golang-slice-struct](https://img.draveness.me/2019-02-20-golang-slice-struct.png)

Go 语言中包含三种初始化切片的方式：

1. 通过下标的方式获得数组或者切片的一部分，特别注意的是 a[low:high] 中 high 的上限，对于数组或者字符串是 `len(a)` ，而对于切片的切片（对切片在执行切片表达式，即 a 原本就是切片），上限是 `cap(a)`。
2. 使用字面量初始化新的切片；
3. 使用关键字 `make` 创建切片；一般只传两个参数，如果是三个则是 `make([]T, size, cap)`。

```go
arr[0:3] or slice[0:3]    // 0 <= 索引值 < 3
slice := []int{1, 2, 3}
slice := make([]int, 10)
```

### 切片的几个注意点

1. 零切片和空切片的关系
   
   ```go
   // 空切片
   var emptySlice1 []int                        // 空切片
   var emptySlice2 []int = make([]int, 0)       // 零切片
   var emptySlice3 []int = []int{}              // 零切片
   
   fmt.Println(emptySlice1 == nil)       // true
   fmt.Println(emptySlice2 == nil)       // false
   fmt.Println(emptySlice3 == nil)       // false
   
   fmt.Println(len(emptySlice1) == 0)    // true
   fmt.Println(len(emptySlice2) == 0)    // true
   fmt.Println(len(emptySlice3) == 0)    // true
   
   fmt.Println(cap(emptySlice1) == 0)    // true
   fmt.Println(cap(emptySlice2) == 0)    // true
   fmt.Println(cap(emptySlice3) == 0)    // true
   ```
   
   从上面看出，零切片和空切片的 `len()` 和 `cap()` 都是0，但只有空切片是 `nil`。同样，使用 `for-range` 遍历都不会产生 `panic`。因此，**判断切片是否为空，始终使用`len() == 0`**。

2. 切片的遍历和数组一样
   
   特别注意的是因为因为 `Go` 语言是采用值拷贝，所以利用 `for-range` 循环遍历时只能作为值的读取而不能对其进行更改，如果想要进行更改则采用基础的 `for` 循环。

3. 切片元素的增加与删除
   
   Go 语言的内建函数`append()`可以为切片动态添加元素。 可以一次添加一个元素，可以添加多个元素，也可以添加另一个切片中的元素（后面加…）。通过var声明的零值切片可以在`append()`函数直接使用，无需初始化。
   
   ```go
   func main(){
       var s []int
       s = append(s, 1)        // [1]
       s = append(s, 2, 3, 4)  // [1 2 3 4]
       s2 := []int{5, 6, 7}  
       s = append(s, s2...)    // [1 2 3 4 5 6 7]
   }
   ```
   
   每个切片会指向一个底层数组，这个数组的容量够用就添加新增元素。当底层数组不能容纳新增的元素时，切片就会自动按照一定的策略进行“扩容”，此时该切片指向的底层数组就会更换。“扩容”操作往往发生在`append()`函数调用时，所以我们通常都需要用原变量接收 `append` 函数的返回值。
   
   **利用 `make` 构造切片，如果指定长度不为0，则会将对应长度填上默认值，如果继续 `append()` 会加入新的值。**

```go
// 完成下面的操作后，a 里面有四个元素[0, 0, 1, 2]
var a = make([]int, 2)
a = append(a, 1)
a = append(a, 2)
```

   Go 语言中并没有删除切片元素的专用方法，我们可以使用切片本身的特性来删除元素。 代码如下：

```go
func main() {
    // 从切片中删除元素
    a := []int{30, 31, 32, 33, 34, 35, 36, 37}
    // 要删除索引为2的元素
    a = append(a[:2], a[3:]...)
    fmt.Println(a) //[30 31 33 34 35 36 37]
}
```

   总结一下就是：要从切片a中删除索引为`index`的元素，操作方法是`a = append(a[:index], a[index+1:]...)`

        

## 哈希表 (map)

哈希表是一种无序的基于 `k-v` 的数据结构，Go 语言中它是**引用类型， 必须初始化才能使用**。map 可以在申明时就填充元素，也可以在后面再填充。

```go
map[KeyType]ValueType

// 申明时就填充
aMap := map[int]string{
    1: "a",
    2: "b",
}

// 构造后填充，此时要确保长度足够
bMap := make(map[int]string, 0)
bMap[1] = "a"


// 这样是不行的，没有初始化
var cMap map[int]string
cMap[1] = "a"
```

map 通过 `for-range` 遍历可以得到对应的 `k-v`，得到的顺序是随机的。

### 哈希表的元素判断和删除

Go 中有个判断 map 中键是否存在的特殊写法，如下，如果返回的第二个参数是 `True` 说明存在，反之不存在。

```go
value, ok := map[key]
```

使用`delete()`内建函数从map中删除一组键值对，`delete()`函数的格式如下，其中，

- map:表示要删除键值对的map
- key:表示要删除的键值对的键

```go
delete(map, key)
```

## 字符串 (string)

Go 语言中的字符串是以原生类型出现的，内部使用 `UTF-8` 编码。字符串的值在 `""` 里面，如果需要定义一个多行字符串必须使用反引号。

```go
str1 := "hello world"
str2 := `hello
world`
```

字符串的常见操作有：

| 方法                                       | 介绍      |
| ---------------------------------------- | ------- |
| `len(str)`                               | 求字符串长度  |
| `fmt.Sprintf` 或者 +                       | 拼接字符串   |
| `strings.Split`                          | 分割字符串   |
| `strings.contatins`                      | 判断是否包含  |
| `strings.HasPrefix`, `strings.HasSuffix` | 前缀/后缀判断 |
| `strings.Index()`                        | 子串出现的位置 |
| `strings.Join(a []string, sep string)`   | 字符串数组拼接 |

### byte 和 rune 类型

组成字符串的元素叫做字符，用 `''` 表示。Go 语言的字符有以下两种类型：

- `unit8` 类型，即 `byte` 类型，代表 `ACSII` 码的一个字符

- `rune` 类型，代表一个 `UTF-8` 字符。一个 `rune` 字符由一个或者多个 `byte` 组成。

需注意，如果要修改字符串，需要先将其转换成 `[]rune` 或者 `[]byte` ，完成后再转换成 string 。
