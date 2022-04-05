# Go 语言中的易错点

参考文档

[go语言坑之for range - Go语言中文网](https://studygolang.com/articles/9701)

----

## for-range 的使用

Go 只提供了一种循环方式，即for循环，在使用时可以像 C 那样使用，也可以通过 `for range` 方式遍历容器类型如数组、切片和映射。但是在使用 `for range` 时，如果使用不当，就会出现一些问题，导致程序运行行为不如预期。例如下面这个例子：

```go
func main() {
    nums := []int{0, 1, 2, 3}
    myMap1 := make(map[int]*int, 0)

    for index, value := range nums {
        myMap1[index] = &value
    }
    fmt.Println("=====myMap1=====")
    printMap(myMap1)
}

func printMap(myMap map[int]*int) {
    for key, value := range myMap {
        fmt.Printf("map[%v]=%v\n", key, *value)
    }
}
```

运行程序可以发现最后的打印结果是

```go
=====myMap1=====
map[0]=3
map[1]=3
map[2]=3
map[3]=3
```

由输出可以知道，映射的值都相同且都是3。其实可以猜测映射的值都是同一个地址，遍历到切片的最后一个元素3时，将3写入了该地址，所以导致映射所有值都相同。其实真实原因也是如此，**因为 `for range` 创建了每个元素的副本，而不是直接返回每个元素的引用。在迭代时，返回的变量是一个迭代过程中依次赋值的新变量，它的地址总是相同的**。如果使用该值变量的地址作为指向每个元素的指针，就会导致结果不如预期。

如果需要解决，常见的有两种做法，一个是对 `for range` 做更改，另一个是使用最普通的 `for` 循环，都会得到最终的结果。

```go
func main() {
    nums := []int{0, 1, 2, 3}
    myMap1 := make(map[int]*int, 0)
    myMap2 := make(map[int]*int, 0)

    for index, value := range nums {
        num := value
        myMap1[index] = &num
    }
    fmt.Println("=====myMap1=====")
    printMap(myMap1)

    for i := 0; i < len(nums); i++ {
        myMap2[i] = &nums[i]
    }
    fmt.Println("=====myMap2=====")
    printMap(myMap2)
}
```
