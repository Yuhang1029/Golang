# Go 中 init() 用法

**参考文档**

[Understanding the init Function in Go](https://www.developer.com/languages/inti-function-golang/)

[The Go init Function | TutorialEdge.net](https://tutorialedge.net/golang/the-go-init-function/)

&emsp;

## 定义

在 Go 中 main() 函数是程序的入口函数，在 `main()` 函数执行完成后程序就结束了。`init()` 函数提供一个特殊的功能，它是程序中最先被执行的，在 `main()` 函数之前，实现包级别的一些初始化工作。`init()` 函数不需要传入参数，也不需要任何返回值，它没有声明，因此也无法引用和外部直接调用。

&emsp;

## 执行顺序

Go 程序在初始化的时候，会对引入的包进行初始化，没有依赖的包被最先初始化。对每一个包来说，最先初始化包作用域的变量，然后会执行 `init()` 函数。在同一个包下允许存在多个 `init()` 函数。**对于不同文件下的 `init()` 函数，如果不存在依赖关系，会按照文件名的字母顺序执行**。这里需要注意的是，当文件名发生更改时，执行顺序也会更改，可能会出现意想不到的结果，所以要避免这种使用。在同一个文件中，`init()` 的执行顺序和声明顺序有关，写在前面的会被先执行。**一个包无论被引入多少次，`init()` 函数只会执行一次**。
