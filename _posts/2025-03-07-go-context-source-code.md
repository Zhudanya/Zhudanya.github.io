---
title: "Go Context源码解析"
date: 2025-03-07
categories: [Go, 源码分析]
tags: [go, context, source-code]
---

# context

context的主要作用是在异步场景中用于实现并发协调以及对 goroutine 的生命周期控制；同时兼备数据存储功能；

> 注：以下所有源码都是基于go1.23.1

## 1 核心数据结构

### 1.1 context.Context

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)  // 返回 context 的过期时间

	Done() <-chan struct{} // 返回用以标识 ctx 是否结束的 chan

	Err() error // 返回 ctx 的错误

	Value(key any) any // 返回 ctx 存放的对应于 key 的 value
}
```

*   Deadline() 方法返回该 context 应该被取消的**截止时间**，如果此 context 没有设置截止时间，则返回的`ok` 值为`false`。
*   Done() 返回一个只读 chan 表示 ctx 是否被取消，当 context 被取消时，此 channel 会被 close 掉。
*   Err() 返回 ctx 的错误原因，如果 ctx 未取消返回 nil；如果调用 cancel() 主动取消了 ctx，返回 Canceld 错误；如果是截止时间到了自动取消了 ctx，返回DeadlineExceeded 错误。
*   Value() 返回与给定键 key 关联的值 value；如果对应 key 没有 value，则返回 nil。

### 1.2 Context.Err()

```go
// Canceled is the error returned by [Context.Err] when the context is canceled.
var Canceled = errors.New("context canceled")

// DeadlineExceeded is the error returned by [Context.Err] when the context's
// deadline passes.
var DeadlineExceeded error = deadlineExceededError{}

type deadlineExceededError struct{}

func (deadlineExceededError) Error() string   { return "context deadline exceeded" }
func (deadlineExceededError) Timeout() bool   { return true }
func (deadlineExceededError) Temporary() bool { return true }
```

*   Canceled：context 被 cancel 时会报此错误；
*   DeadlineExceeded：context 超时时会报此错误

## 2 Context的实现

### 2.1 emptyCtx

`emptyCtx` 是最基础的 context 实现，定义如下：

```go
// An emptyCtx is never canceled, has no values, and has no deadline.
// It is the common base of backgroundCtx and todoCtx.
type emptyCtx struct{}

func (emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (emptyCtx) Done() <-chan struct{} {
	return nil
}

func (emptyCtx) Err() error {
	return nil
}

func (emptyCtx) Value(key any) any {
	return nil
}
```

emptyCtx作为其他 context 实现的“基类”，它并没有控制链路的能力，也没有安全传值的功能，只是表明了语义，它通常作为整个 context 链路的起点；

#### 2.1.1 context.Background() & context.TODO()

```go
type backgroundCtx struct{ emptyCtx }

func (backgroundCtx) String() string {
	return "context.Background"
}

type todoCtx struct{ emptyCtx }

func (todoCtx) String() string {
	return "context.TODO"
}

//---------------------------------------
func Background() Context {
	return backgroundCtx{}
}

func TODO() Context {
	return todoCtx{}
}
```

我们所常用的 context.Background() 和 context.TODO() 方法返回的均是 emptyCtx 类型的一个实例；它们是整个 context 链路的基础。

### 2.2 cancelCtx

`cancelCtx` 结构体定义如下：

```go
type cancelCtx struct {
	Context // 继承的父 Context

	mu       sync.Mutex            // 持有锁保护下面这些字段
	done     atomic.Value          // 值为 chan struct{} 类型，会被惰性创建，在第一次调用取消函数 cancel() 时被关闭，表示 Context 已被取消
	children map[canceler]struct{} // 所有可以被取消的子 Context 集合，它们在第一次调用取消函数 cancel() 时被级联取消，然后置为 nil
	err      error
	cause    error
}

type canceler interface {
	cancel(removeFromParent bool, err, cause error)
	Done() <-chan struct{}
}
```

*   Deadline方法

    cancelCtx没重写Deadline()方法，若调用cancelCtx的Deadline()则会执行到父节点的对应方法；

*   Done方法

    ```go
    func (c *cancelCtx) Done() <-chan struct{} {
    	d := c.done.Load()
    	if d != nil {
    		return d.(chan struct{})
    	}
    	c.mu.Lock()
    	defer c.mu.Unlock()
    	d = c.done.Load()
    	if d == nil {
    		d = make(chan struct{})
    		c.done.Store(d)
    	}
    	return d.(chan struct{})
    }
    ```

    懒加载，只有在调用Done()时才会初始化c.done并返回；

*   Err方法

    ```go
    func (c *cancelCtx) Err() error {
    	c.mu.Lock()
    	err := c.err
    	c.mu.Unlock()
    	return err
    }
    ```

    Err()可能也会被并发调用，所以需要加锁；

*   Value 方法

    ```go
    func (c *cancelCtx) Value(key any) any {
    	if key == &cancelCtxKey {
    		return c
    	}
    	return value(c.Context, key)
    }
    ```

**context.WithCancel() 方法：**

WithCancel() 是context包开放的一个创建cancelCtx对象的方法；

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := withCancel(parent)
	return c, func() { c.cancel(true, Canceled, nil) }
}

func withCancel(parent Context) *cancelCtx {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := &cancelCtx{}
	c.propagateCancel(parent, c)
	return c
}
```

`propagateCancel`  方法主要做目的是：做父节点和子节点的关联，保证父节点被取消了，子节点也要被取消

重点关注下 `propagateCancel`  的实现：

```go
func (c *cancelCtx) propagateCancel(parent Context, child canceler) {
	c.Context = parent

    // ①
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

    // ② 
	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err(), Cause(parent))
		return
	default:
	}

    // ③
	if p, ok := parentCancelCtx(parent); ok {
		// parent is a *cancelCtx, or derives from one.
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err, p.cause)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
		return
	}

    // ④
	if a, ok := parent.(afterFuncer); ok {
		// parent implements an AfterFunc method.
		c.mu.Lock()
		stop := a.AfterFunc(func() {
			child.cancel(false, parent.Err(), Cause(parent))
		})
		c.Context = stopCtx{
			Context: parent,
			stop:    stop,
		}
		c.mu.Unlock()
		return
	}

    // ⑤
	goroutines.Add(1)
	go func() {
		select {
		case <-parent.Done():
			child.cancel(false, parent.Err(), Cause(parent))
		case <-child.Done():
		}
	}()
}
```

*   ① 先检查父节点 `done := parent.Done()`，如果该节点的父亲是永远不会取消的类型，比如：emptyCtx，则直接return
*   ② 多路复用监听父节点的done，若父节点已经被取消了，则父节点下关联的所有子节点也需要被取消，然后return即可
*   ③ 如果当前节点的父节点也是 `cancelCtx` 则判断下，若父节点已取消了则同样取消当前节点，若没取消则只需要将当前节点加到父节点的 children 集合中
*   ④ 先搁置。。。
*   ⑤ 新创建一个goroutine然后阻塞监听父节点的Done，若父节点有生命周期终止消息通知来，然后同时处理子节点的取消逻辑；新加一个 `case <-child.Done():` 的目的是：若子节点已经先于父节点被取消了则这个守护协程就可以退出了；通过这样的方式来实现父到子的生命周期终止的单向消息传递

cancel 方法：

```go
func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) { 
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	if cause == nil {
		cause = err
	}
    
    // ①
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	c.cause = cause
    
    // ②
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
    
    // ③
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err, cause)
	}
	c.children = nil
	c.mu.Unlock()

    // ⑤
	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

*   ① 检查 cancelCtx.err 是否已被赋值，已被赋值说明当前这个节点已被终止
*   ② 懒加载机制，若 cancelCtx.done 从来没有被调用声明过，则直接给它一个全局的已被关闭的标识；若有则直接关闭这个chan，这样上游所有调用Done()的地方就不会被阻塞了
*   ③ 负责将当前节点的每个子节点都取消终止掉
*   根据传入的 removeFromParent 判断是否需要把当前节点从父节点的子节点集合中移除

### 2.3 timerCtx

`timerCtx` 结构体定义如下：

```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```

timerCtx 在 cancelCtx 基础上又做了一层封装，除了继承 cancelCtx 的能力之外，新增了一个 time.Timer 用于定时终止 context；另外新增了一个 deadline 字段用于 timerCtx 的过期时间；

`timerCtx` 只实现了 `Deadline()`  方法的封装：

```go
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}
```

context.Context interface 下的 Deadline api 仅在 timerCtx 中有效，由于展示其过期时间；

`timerCtx` 也重写了 cancelCtx 的 cancel() 方法：

```go
func (c *timerCtx) cancel(removeFromParent bool, err, cause error) {
	c.cancelCtx.cancel(false, err, cause)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```

*   复用父节点 `cancelCtx` 的 cancel
*   然后再根据 `timerCtx` 的特性将 timer 定时器关闭掉（因为当前这个timerCtx被手动取消了，所以要回收这个资源）

**context.WithTimeout() 和 context.WithDeadline()方法：**

如何去创建 `timerCtx` 呢，context包暴露了两个方法：`WithTimeout()` 和 `WithDeadline()` ：

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) { // 传入的是持续时间
	return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) { // 传入的是结束时间戳
	return WithDeadlineCause(parent, d, nil)
}
```

最终都是调用 `WithDeadlineCause` ：

```go
func WithDeadlineCause(parent Context, d time.Time, cause error) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
    
    // ①
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		deadline: d,
	}
    // ② 
	c.cancelCtx.propagateCancel(parent, c)
    
    // ③
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded, cause) // deadline has already passed
		return c, func() { c.cancel(false, Canceled, nil) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
    
    // ④
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded, cause)
		})
	}
	return c, func() { c.cancel(true, Canceled, nil) }
}
```

*   ① 要保证设置的过期时间是成立的，不能比父节点的过期时间来的更晚（父ctx会早于子ctx被取消掉，那么这次创建就是无意义的）
*   ② `propagateCancel` 上面分析过
*   ③ 如果已到期，则需要取消当前节点
*   ④ 设置一个定时器，到取消时间后执行 cancel 方法取消当前节点

### 2.4 valueCtx

具有数据存储功能的节点；

`valueCtx` 结构体定义如下：

```go
type valueCtx struct {
	Context
	key, val any
}
```

可以看到每个 `valueCtx` 只有一组key，val对；所以如果想创建多组键值对的话需要多个 `valueCtx` 串联起来；

这也是 `valueCtx` 并发安全的保证，保证绝不修改现有的 context 对象；看看树状图：

![E-Context-valueCtx树状图.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/59d5aa0adca44baab6e7e742f0056f33~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgX-aWsOS4gA==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMjM1MzQxODcyNjgxMDg1NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1774948260&x-orig-sign=Cb9Q8WY%2F9dHCjEN%2FUE5tU9jHiGA%3D)

Value 方法：

```go
func (c *valueCtx) Value(key any) any {
	if c.key == key {
		return c.val
	}
	return value(c.Context, key)
}
```

如果要查找的key正好是当前的节点，则直接返回对应的val即可，否则调用 value() 方法从 parent context 中依次查找：

```go
func value(c Context, key any) any {
	for {
		switch ctx := c.(type) {
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
		case withoutCancelCtx:
			if key == &cancelCtxKey {
				// This implements Cause(ctx) == nil
				// when ctx is created using WithoutCancel.
				return nil
			}
			c = ctx.c
		case *timerCtx:
			if key == &cancelCtxKey {
				return &ctx.cancelCtx
			}
			c = ctx.Context
		case backgroundCtx, todoCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}
```

以 `case *valueCtx:` 为例，可以看到，是逐层向上查找，直到找到为止，若这条链路上一直没有对应的节点则会找到根节点上 `case backgroundCtx, todoCtx:` ，则直接 `return nil` 即可；

valueCtx用法总结：

*   一个 valueCtx 只能存储一个键值对，因此如果要存储多个键值对则需要建立一个类似链表的结构，会造成空间资源浪费，而且查找复杂度也是O(N)；
*   根据 valueCtx 的特性，使用该节点存储的键值对无法支持基于key的去重；
*   因此该 `valueCtx` 只适合存放少量作用域较大的全局数据；

**context.WIthValue() 方法：**

```go
func WithValue(parent Context, key, val any) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```