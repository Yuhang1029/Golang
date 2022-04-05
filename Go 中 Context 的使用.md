# Go 中 Context 的使用

**参考文档**

 [Go 语言并发编程与 Context | Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context/)

[Go Concurrency Patterns: Context - The Go Programming Language](https://go.dev/blog/context)

[How to Correctly use context.Context in Go 1.7](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39)

[Go 如何正确使用 Context - 掘金](https://juejin.cn/post/6844903929340231694)

## 什么是上下文 Context

### 为什么需要上下文

在 `Goroutine` 构成的树形结构中对信号进行同步以减少计算资源的浪费是 `context.Context` 的最大作用。Go 服务的每一个请求都是通过单独的 `Goroutine` 处理的，HTTP/RPC 请求的处理器会启动新的 `Goroutine` 访问数据库和其他服务。我们可能会创建多个 `Goroutine` 来处理一次请求，而 `context.Context` 的作用是在不同 `Goroutine` 之间同步请求特定数据、取消信号以及处理请求的截止日期。

从下面两张图可以看出，第一张图里面，当最上层的 `Goroutine` 因为某些原因执行失败时，下层的 `Goroutine` 由于没有接收到这个信号所以会继续工作；但是当我们正确地使用 `context.Context` 时，就可以在下层及时停掉无用的工作以减少额外资源的消耗。

![golangwithoutcontext](https://img.draveness.me/golang-without-context.png)

![golang-with-context](https://img.draveness.me/golang-with-context.png)

### Context 介绍

上下文 [`context.Context`](https://draveness.me/golang/tree/context.Context) Go 语言中用来设置截止日期、同步信号，传递请求相关值的结构体。上下文与 `Goroutine` 有比较密切的关系，是 Go 语言中独特的设计，在其他编程语言中我们很少见到类似的概念。

`context.Context` 是 Go 语言在1.7版本中引入标准库的接口，里面定义了四种方法。

```go
// A Context carries a deadline, cancellation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

其中包括：

1. `Deadline` — 返回 `context.Context` 被取消的时间，也就是完成工作的截止日期；
2. `Done` — 返回一个 `Channel`，这个 `Channel` 会在当前工作完成或者上下文被取消后关闭，多次调用 `Done` 方法会返回同一个 `Channel`；它仅用来接收，永远不会用于发送。如果该方法返回的 `Channel` 可以读取，则意味着 parent context 已经发起了取消请求，我们通过 `Done` 方法收到这个信号后，就应该做清理操作，然后退出 `goroutine`，释放资源。
3. `Err` — 返回 `context.Context` 结束的原因，它只会在 `Done` 方法对应的 `Channel` 关闭时返回非空的值；
   1. 如果 `context.Context` 被取消，会返回 `Canceled` 错误；
   2. 如果 `context.Context` 超时，会返回 `DeadlineExceeded` 错误；
4. `Value` — 从 `context.Context` 中获取键对应的值，对于同一个上下文来说，多次调用 `Value` 并传入相同的 `Key` 会返回相同的结果，该方法可以用来传递请求特定的数据；

### 并发控制

Context 的引入使得并发控制变得更加简单，这也是引入的初衷。

```go
func main() {
        ctx, cancel := context.WithCancel(context.Background())
        go watch(ctx,"【监控1】")
        go watch(ctx,"【监控2】")
        go watch(ctx,"【监控3】")

        time.Sleep(10 * time.Second)
        fmt.Println("可以了，通知监控停止")
        cancel()
        //为了检测监控过是否停止，如果没有监控输出，就表示停止了
        time.Sleep(5 * time.Second)
}

func watch(ctx context.Context, name string) {
        for {
                select {
                case <-ctx.Done():
                        fmt.Println(name,"监控退出，停止了...")
                        return
                default:
                        fmt.Println(name,"goroutine监控中...")
                        time.Sleep(2 * time.Second)
                }
        }
}
```

在 `goroutine` 中，使用 select 调用 `<-ctx.Done()` 判断是否要结束，如果接受到值的话，就可以返回结束 `goroutine` 了；如果接收不到，就会继续进行监控。示例中启动了3个监控 `goroutine` 进行不断的监控，每一个都使用了`Context` 进行跟踪，当我们使用 `cancel` 函数通知取消时，这3个 `goroutine` 都会被结束。

## Context 的派生

官方文档原文：

> The context package provides functions to derive new Context values from existing ones. These values form a tree: when a Context is canceled, all Contexts derived from it are also canceled.

### 设计原理

对信号进行同步以减少计算资源的浪费是 Context 的最大作用。需要注意，上下文 Context 的不断衍生都是一个树形结构，其根节点不会改变。后面会讲到，**如果取消一个节点，会顺带取消掉所有的子节点（从上至下）；而当进行值的查找的时候是一个回溯的过程（从下至上）**。

### 默认上下文

`context` 包中最常使用的还是 `context.Background` 和 `context.TODO` ，这两个方法都会返回预先初始化好的私有变量 `background` 和 `todo`，它们会在同一个 Go 程序中被复用

```go
// Background returns an empty Context. It is never canceled, has no deadline,
// and has no values. Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```

这两个私有变量都是通过 `new(emptyCtx)` 语句初始化的，它们是指向私有结构体 `context.EmptyCtx` 的指针，这是最简单、最常用的上下文类型，它实际上没有任何功能。

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (*emptyCtx) Done() <-chan struct{} {
    return nil
}

func (*emptyCtx) Err() error {
    return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
    return nil
}
```

从源代码来看，`context.Background` 和 `context.TODO` 也只是互为别名，没有太大的差别，只是在使用和语义上稍有不同：

- `context.Background` 是上下文的默认值，所有其他的上下文都应该从它衍生出来；
- `context.TODO` 应该仅在不确定应该使用哪种上下文时使用，例如你没有 `Context`，却需要调用一个有 `Context` 的函数的话，可以使用它，注意不要使用 nil。

在多数情况下，如果当前函数没有上下文作为入参，我们都会使用`context.Background` 作为起始的上下文向下传递。

### 构造新的上下文

#### 取消信号

`context.WithCancel` 函数能够从 `context.Context` 中衍生出一个新的子上下文并返回用于取消该上下文的函数。一旦我们执行返回的取消函数，当前上下文以及它的子上下文都会被取消，所有的 `Goroutine` 都会同步收到这一取消信号。

```go
// WithCancel returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed or cancel is called.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}

// A CancelFunc cancels a Context.
type CancelFunc func()
```

- `context.newCancelCtx` 将传入的上下文包装成私有结构体 `context.cancelCtx` ；
- `context.propagateCancel` 会构建父子上下文之间的关联，当父上下文被取消时，子上下文也会被取消。`context.propagateCancel` 的作用是在 `parent` 和 `child` 之间同步取消和结束的信号，保证在 `parent` 被取消时，`child` 也会收到对应的信号，不会出现状态不一致的情况。
* `CancelFunc` 取消函数，该函数可以取消一个`Context`，以及这个节点 `Context` 下所有的所有的 `Context`，不管有多少层级。当返回的取消函数被调用时，或者父 `Context` 的 `Done` 通道被关闭时，返回的 `Context` 的 `Done` 通道将被关闭，顺序以最先发生的为准。

`context.WithTimeOut` 函数接受一个 `Context` 和超时时间作为参数，返回其子 `Context` 和取消函数 `cancel`。相比上面的多了一个时间作为参数。`context.WithTimeOut` 实际上使用的是就是 `context.WithDeadline`。

```go
// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
   return WithDeadline(parent, time.Now().Add(timeout))
}

// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {}
```

#### 传值方法

`context` 中的 `context.WithValue` 能从父上下文中创建一个子上下文，传值的子上下文使用 `context.ValueCtx` 类型

```go
// WithValue returns a copy of parent whose Value method returns val for key.
func WithValue(parent Context, key, val interface{}) Context {
    if key == nil {
        panic("nil key")
    }
    if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}
```

`context.ValueCtx` 结构体只会相应的 `Value()` 方法。

```go
type valueCtx struct {
    Context
    key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

如果 `context.ValueCtx` 中存储的键值对与`context.ValueCtx.Value`方法中传入的参数不匹配，就会从父上下文中查找该键对应的值直到某个父上下文中返回 `nil` 或者查找到对应的值。

需要注意的是为了 key 的类型，为了避免不同包使用时产生的潜在冲突，不要使用类似 string 或者 int 等内置的类型，最推荐的方法就是自己定义一个类型。例如下面这个例子。

```go
const (
   currentVersion = "1.0.5"
)

type (
   solveCtxKey struct{}
)

func WithSolveContext(ctx context.Context) context.Context {
   return context.WithValue(ctx, solveCtxKey{}, &SolveContext{
      Version: currentVersion,
   })
}

func GetSolveContext(ctx context.Context) *SolveContext {
   if ctx.Value(solveCtxKey{}) == nil {
      return nil
   }
   v := ctx.Value(solveCtxKey{}).(*SolveContext)
   return v
}
```

这里使用 WithXxx 而不是 SetXxx 也是因为 `Context` 实际上是不可改变的，所以不是修改 `Context` 里某个值，而是产生新的 `Context` 带某个值。这个同样遵循前面说的树的结构。`Context.Value` 是不可改变的，不要试图在 `Context.Value` 里存某个可变更的值，然后改变，期望别的 `Context` 可以看到这个改变。

## 如何正确使用 Context

### 两个基本的原则

1. 使用范围
   
   需要确保 Context 的使用维度是在请求范围上。例如可以是一次数据库的查询请求而不是伴随一个数据库。例如下面这个例子。
   
   ```go
   func Solve(ctx context.Context, request *api.TagSolveProblemRequest) (resp *api.TagSolveProblemResponse, err error){}
   ```

2. 保持流动性
   
    `Context` 不应该像一个结构体一样被存储，而是应该作为一个接口顺着方法的调用一层一层传下去。理想情况下，`Context` 存在于调用栈（Call Stack）中，随着请求的开始而被创建，随着请求的结束而到期销毁。

### Context 的缺点

前面提到的 `Context` 的使用维度应该是在请求层面上，并且我们意识到似乎所有后面需要的内容都是从请求中获取的，那么如果针对函数舍弃所有的参数而把需要的内容都放在 `Context` 里会怎样呢？

```go
func IsAdminUser(ctx context.Context) bool {
  x := token.GetToken(ctx)
  userObject := auth.AuthenticateToken(x)
  return userObject.IsAdmin() || userObject.IsRoot()
}

func IsAdminUser(token string, authService AuthService) int {
  userObject := authService.AuthenticateToken(token)
  return userObject.IsAdmin() || userObject.IsRoot()
}
```

如果是采用第一个方法，它的入参只有一个 `Context`，但是使用者并不清楚里面究竟有什么内容。由此可见， `Context` 并不是一个万能百宝箱，适合把任何需要的信息都放进去。实际上，它让你的 API 定义更加模糊。

同时，Context传递的数据中key、value都是interface类型，这种类型编译期无法确定类型，所以不是很安全。

### 什么时候使用最合适

> Inform, not control.The content of context.Value is for maintainers not users. It should never be required input for documented or expected results.

主要是一种通知类型的信息而不是控制类型的，如果你发现你的函数在某些 `Context.Value` 下无法正确工作，那就说明这个 `Context.Value` 里的信息不应该放在里面，而应该放在接口上。因为已经让接口太模糊了。

目前看到的文档中，比较明显的适合使用 `Context.Value` 就是 Request ID，并且仅仅是作为日志或者记录使用而不会影响整个程序的运行逻辑。如果是确实需要使用，也尽量减少使用的频率，集中使用的地方。

## 一些好的习惯

1. 如果函数的参数中有 Context，将其作为第一个变量，而不是一个函数中间的某一个。
   
   > At Google, we require that Go programmers pass a Context parameter as the first argument to every function on the call path between incoming and outgoing requests.

2. 要养成关闭 `Context` 的好习惯，特别是超时的 `Context` 。在建立之后，立即 defer `cancel()` 是一个好习惯。
   
   ```go
   ctx, cancel := context.WithTimeout(parentCtx, time.Second * 2)
   defer cancel()
   ```

3. 对于需要传递 `Context` 的地方，如果不确定如何使用不要传入 nil，而是 `context.TODO`。

4. 使用 Context 进行传递参数请求的所有参数一种非常差的设计，比较常见的使用场景是传递请求对应用户的认证令牌以及用于进行分布式追踪的请求 ID。
