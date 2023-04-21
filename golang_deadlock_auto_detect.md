# golang deadlock 死锁检测的实现原理

😅 以前我写过一篇如何排查死锁的问题，简单说就是分析输出的所有协程的函数调用栈，前后记录两次调用栈信息，然后用文本脚本去掉干扰信息，没有锁操作的协程，这样就好分析多了。

虽然问题解决了，但我在想是否可以自动化检测golang的死锁冲突。我想到了动态检查死锁的思路，但还没有来得及去实现就歇菜了。朋友在github里找到可用的golang的死锁检查库，我简单过了一遍代码，发现跟我的想法有点像，当然他的更完美。 😅

这个死锁检测库并不是基于静态分析的，而是基于运行时检测的。

代码地址 [https://github.com/sasha-s/go-deadlock](https://github.com/sasha-s/go-deadlock)

## 源码分析

直接看源码。原理很简单，就是获取当前协程的goroutine id，然后存了当前协程没有释放的lock的对象。 这时候当其他协程去lock的时候，会触发prelock检测，检测有没有冲突lock关系。

### go-deadlock 死锁检测实现原理

下面是go-deadlock的存储锁关系的数据结构.

```go
// xiaorui.cc
type lockOrder struct {
	mu    sync.Mutex
	cur   map[interface{}]stackGID // stacktraces + gids for the locks currently taken.
	order map[beforeAfter]ss       // expected order of locks.
}

type stackGID struct {
	stack []uintptr
	gid   int64
}

type beforeAfter struct {
	before interface{}
	after  interface{}
}

type ss struct {
	before []uintptr
	after  []uintptr
}
```

下面是拿锁，释放锁，检测死锁关系的过程

```go
func (m *Mutex) Lock() {
	lock(m.mu.Lock, m)
}

func (m *Mutex) Unlock() {
	m.mu.Unlock()
	if !Opts.Disable {
                // 在map中删除锁关系
		postUnlock(m)
	}
}

func lock(lockFn func(), ptr interface{}) {
        // 预先死锁检测
	preLock(4, ptr)
	if Opts.DeadlockTimeout &lt;= 0 {
		lockFn()
	} else {
                // 开启死锁超时检测
		ch := make(chan struct{})
		go func() {
			for {
				t := time.NewTimer(Opts.DeadlockTimeout)
				defer t.Stop() // This runs after the losure finishes, but it's OK.
				select {
				case &lt;-t.C:
					lo.mu.Lock()
					prev, ok := lo.cur[ptr]
					if !ok {
						lo.mu.Unlock()
						break // Nobody seems to be holding the lock, try again.
					}
					Opts.mu.Lock()
					...
					stacks := stacks()
					grs := bytes.Split(stacks, []byte("\n\n"))
					for _, g := range grs {
						if goid.ExtractGID(g) == prev.gid {
							fmt.Fprintln(Opts.LogBuf, "Here is what goroutine", prev.gid, "doing now")
							Opts.LogBuf.Write(g)
							fmt.Fprintln(Opts.LogBuf)
						}
					}
					Opts.mu.Unlock()
					lo.mu.Unlock()
					Opts.OnPotentialDeadlock()
					&lt;-ch
					return
				case &lt;-ch:
					return
				}
			}
		}()
		lockFn()
                // 锁检测收尾操作
		postLock(4, ptr)
		return
	}
	postLock(4, ptr)
}

func (l *lockOrder) preLock(skip int, p interface{}) {
    if Opts.DisableLockOrderDetection {
        return
    }
    stack := callers(skip)
    gid := goid.Get()
    l.mu.Lock()
    for b, bs := range l.cur {
        if b == p {
            // 还没有释放的锁的协程id是否跟当前id一样
            if bs.gid == gid {
                Opts.mu.Lock()
                fmt.Fprintln(Opts.LogBuf, header, "Recursive locking:")
                fmt.Fprintf(Opts.LogBuf, "current goroutine %d lock %p\n", gid, b)
                printStack(Opts.LogBuf, stack)
                fmt.Fprintln(Opts.LogBuf, "Previous place where the lock was grabbed (same goroutine)")
                printStack(Opts.LogBuf, bs.stack)
                l.other(p)
                if buf, ok := Opts.LogBuf.(*bufio.Writer); ok {
                    buf.Flush()
                }
                Opts.mu.Unlock()
                Opts.OnPotentialDeadlock()
            }
            continue
        }
        if bs.gid != gid { // We want locks taken in the same goroutine only.
            continue
        }
        // 查看是否有交叉拿锁的现象
        if s, ok := l.order[beforeAfter{p, b}]; ok {
            ...
            Opts.mu.Unlock()
            Opts.OnPotentialDeadlock()
        }

        // 存储beforeAfter的关系
        l.order[beforeAfter{b, p}] = ss{bs.stack, stack}
        if len(l.order) == Opts.MaxMapSize { // Reset the map to keep memory footprint bounded.
            l.order = map[beforeAfter]ss{}
        }
    }
    l.mu.Unlock()
}

func (l *lockOrder) postUnlock(p interface{}) {
	l.mu.Lock()
	delete(l.cur, p)
	l.mu.Unlock()
}
```

这里的协程id是使用 github.com/petermattis/goid 来获取的，该库使用cgo的方法来获取id.

### 死锁检测的使用?

go deadlock接口上兼容了标准库sync.Mutex，另外deadlock也实现了读写锁。

```go
// xiaorui.cc
package main

import (
    "fmt"
    "sync"
    "time"

    "github.com/sasha-s/go-deadlock"
)

// xiaorui.cc

var (
    mu1 deadlock.Mutex
    mu2 deadlock.Mutex
    wg sync.WaitGroup
)

func main() {
    wg.Add(2)

    go func() {
        mu1.Lock()
        time.Sleep(1 * time.Second)
        mu2.Lock()
    }()

    go func() {
        mu2.Lock()
        mu1.Lock()
    }()

    go func() {
        for {
            time.Sleep(1 * time.Second)
            fmt.Println("xiaorui.cc")
        }
    }()

    wg.Wait()
}
```

错误信息如下:

```c
// xiaorui.cc
POTENTIAL DEADLOCK: Inconsistent locking. saw this ordering in one goroutine:
happened before
xiaorui.cc
aa.go:28 main.main.func2 { mu2.Lock() } 

happened after
aa.go:29 main.main.func2 { mu1.Lock() }

in another goroutine: happened before
aa.go:22 main.main.func1 { mu1.Lock() } 

happened after
aa.go:24 main.main.func1 { mu2.Lock() } 

Other goroutines holding locks:
goroutine 20 lock 0x11a71b0
aa.go:22 main.main.func1 { mu1.Lock() } 
```

## 在多场景下go-deadlock如何做的死锁检测 ？

### 场景1: 当协程1拿到了lock1的锁，然后再尝试拿lock1锁？

很简单，用一个map存入所有为释放锁的协程id， 当检测到gid相同时, 触发OnPotentialDeadlock回调方法。

如果拿到一个锁，又通过 go func()去拿同样的锁，这时候就无法快速检测死锁了，只能依赖go-deadlock提供了锁超时检测。

### 场景2: 协程1拿到了lock1, 协程2拿到了lock2, 这时候协程1再去拿lock2, 协程2尝试去拿lock1

这是交叉拿锁引起的死锁问题，如何解决？ 我们可以存入beferAfter关系。在go-deadlock里有个order map专门来存这个关系。 当协程1再去拿lock2的时候, 如果order里有 lock1-lock2, 那么触发OnPotentialDeadlock回调方法。

### 场景3: 如果协程1拿到了lock1，但是没有写unlock方法，协程2尝试拿lock1, 会一直阻塞的等待。

go deadlock会针对开启DeadlockTimeout >0 的加锁过程，new一个协程来加入定时器判断是否锁超时。

## 总结:

这个动态死锁检测库当然不能在生产环境中使用了，毕竟来回折腾那几个map存beforeAfter和协程id是有开销的是，当DeadlockTimeout不为0时， 他会new一个协程来再次是否锁超时。如果超时，那么大概率是死锁的。

当检测出现死锁的时候，go-deadlock不仅会打印协程id，而且会输出发生死锁的协程调用栈信息。通过调用栈协议很方便可以找到对应代码。