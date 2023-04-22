# 源码分析 golang context 的超时及关闭实现

Golang的context的作用就不多说了，就是用来管理调用上下文的，控制一个请求的生命周期。golang的context库里有四个组件。 withCancel用来控制取消事件，withDeadline 和 withTimeout 是控制超时，withValue可以传递一些key value。 

```go
// xiaorui.cc

func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

func WithValue(parent Context, key, val interface{}) Context
```

## 源码分析context

下面的结构图就很形象的说明了各个 context 的关联关系。context 节点通过 children map 来连接子 context 节点。总之，context节点是层层关联的。关闭的时候自然也是层层关闭。

![https://xiaorui.cc/wp-content/uploads/2018/12/20181201015259_78633.jpg](https://xiaorui.cc/wp-content/uploads/2018/12/20181201015259_78633.jpg)

WithTimeout 也是通过 WithDeadline 来实现的。

```go
// xiaorui.cc
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

### context WithDeadline 的实现

通过下面的 WithDeadline 方法，我们可以分析出创建一个子 context 及定时器过程。

```go
// xiaorui.cc
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
        ...
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),   // 返回一个子的context
		deadline:  d,
	}
        //  添加父节点和子节点的关联关系
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(true, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
                // 添加定时器
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
        // 返回 context 和 关闭的方法
	return c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}

func propagateCancel(parent Context, child canceler) {
	// ...
	// ...

	} else {
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

timectx 的 cancel 实现.

```go
// xiaorui.cc
func (c *timerCtx) cancel(removeFromParent bool, err error) {
        ...
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
                // 关闭定时器
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}

// removeChild removes a context from its parent.
func removeChild(parent Context, child canceler) {
	p, ok := parentCancelCtx(parent)
	if !ok {
		return
	}
	p.mu.Lock()
	if p.children != nil {
		delete(p.children, child)  // 从父层里删除子节点
	}
	p.mu.Unlock()
}
```

## context cancel 关闭的设计

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
         ...
	c.mu.Lock()
        ...
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done) // 关闭
	}
        // 重点: 层层关闭，从父层到子层，一个个的关闭.
	for child := range c.children {
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

        // 从父context child里删除子context节点
	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

## 总结：

context的源码很简单，代码也精简的，有兴趣的朋友可以细细的琢磨下。