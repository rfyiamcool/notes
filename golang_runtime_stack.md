# golang runtime.stack 加锁引起 runtime 调度阻塞

我们知道在golang社区里多数web框架自带了panic后的recovery功能。go的recovery可以当成一个保护方案，避免因为各种错误导致进程挂掉，业务受到影响。golang echo 这个框架也有 recovery 的功能，但他的默认方法着实坑人，该坑会引起我们标题中的所描述的高延迟和阻塞问题。

## 源码解读

下面是官方给的例子，像其他web框架一样，把recovery做到了中间件里面。

```go
// xiaorui.cc
package main

import (
	"net/http"
	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
)

func main() {
	// Echo instance
	e := echo.New()

	// Middleware
	e.Use(middleware.Recover())

	// ...
	// ...
}
```

默认的config.DisableStackAll为false，下面使用了 ! 符号，负负得正。简单说，默认是打印所有的协程栈，当然最后的打印依赖stackSize，buf值为4KB大小。

```go
// xiaorui.cc
if r := recover(); r != nil {
	err, ok := r.(error)
	if !ok {
		err = fmt.Errorf("%v", r)
	}
	stack := make([]byte, config.StackSize)
	length := runtime.Stack(stack, !config.DisableStackAll)
	if !config.DisablePrintStack {
		c.Logger().Printf("[PANIC RECOVER] %v %s\n", err, stack[:length])
	}
	c.Error(err)
}
```

当 `all=false` 时，只会获取当前协程的函数调用栈信息，无需加锁。但 `all=true` 时，意味着要获取所有协程的栈信息，在go runtime的pmg调度模型下，为了保证并发操作安全，自然就需要在stack方法里加了锁，且锁的粒度还不小，直接调用 stopTheWorld 用来阻塞 GC 的操作。

goroutineheader 方法用来获取协程的状态信息，比如等待锁，scan，已等待时间等。allgs是runtime保存的所有已创建协程的容器，当然不会去追踪已经消亡的协程。另外，为了保护allgs切片的安全，还会对allglock加锁，在allgadd()创建goroutine和checkdead()检测死锁里会产生锁竞争。

```go
// xiaorui.cc

func Stack(buf []byte, all bool) int {
	if all {
		stopTheWorld("stack trace")
	}

	n := 0
	if len(buf) > 0 {
		gp := getg()
		sp := getcallersp()
		pc := getcallerpc()
		systemstack(func() {
			g0 := getg()
			g0.m.traceback = 1
			g0.writebuf = buf[0:0:len(buf)]
			goroutineheader(gp)
			traceback(pc, sp, 0, gp)
			if all {
				tracebackothers(gp)
			}
			g0.m.traceback = 0
			n = len(g0.writebuf)
			g0.writebuf = nil
		})
	}

	if all {
		startTheWorld()
	}
	return n
}

func tracebackothers(me *g) {
	g := getg()
	gp := g.m.curg
	if gp != nil && gp != me {
		print("\n")
		goroutineheader(gp)
		traceback(^uintptr(0), ^uintptr(0), 0, gp)
	}

	lock(&allglock)
	for _, gp := range allgs {
		if gp == me || gp == g.m.curg || readgstatus(gp) == _Gdead || isSystemGoroutine(gp, false) && level < 2 {
			continue
		}
		print("\n")
		goroutineheader(gp)
		if gp.m != g.m && readgstatus(gp)&^_Gscan == _Grunning {
			print("\tgoroutine running on other thread; stack unavailable\n")
			printcreatedby(gp)
		} else {
			traceback(^uintptr(0), ^uintptr(0), 0, gp)
		}
	}
	unlock(&allglock)
}

func goroutineheader(gp *g) {
	// approx time the G is blocked, in minutes
	var waitfor int64
	if (gpstatus == _Gwaiting || gpstatus == _Gsyscall) && gp.waitsince != 0 {
		waitfor = (nanotime() - gp.waitsince) / 60e9
	}
	print("goroutine ", gp.goid, " [", status)
	if isScan {
		print(" (scan)")
	}
	if waitfor >= 1 {
		print(", ", waitfor, " minutes")
	}
...
```

我们可以设想一下，在echo里某个接口并发的出现了recovery的问题，那么都会走上面的加锁的过程，而且还并发操作，那么势必会造成阻塞和高时延的问题。

## 其他web框架的recovery源码

追了下gin和iris的recovery源码实现，都只传递需要打印的栈层数，然后调用runtime.Caller获取栈的信息。[https://github.com/gin-gonic/gin/blob/master/recovery.go](https://github.com/gin-gonic/gin/blob/master/recovery.go)

```go
// xiaorui.cc

func stack(skip int) []byte {
	buf := new(bytes.Buffer) // the returned data
	// As we loop, we open files and read them. These variables record the currently
	// loaded file.
	var lines [][]byte
	var lastFile string
	for i := skip; ; i++ { // Skip the expected number of frames
		pc, file, line, ok := runtime.Caller(i)
		if !ok {
			break
		}
		// Print this much at least.  If we can't find the source, it won't show.
		fmt.Fprintf(buf, "%s:%d (0x%x)\n", file, line, pc)
		if file != lastFile {
			data, err := ioutil.ReadFile(file)
			if err != nil {
				continue
			}
			lines = bytes.Split(data, []byte{'\n'})
			lastFile = file
		}
		fmt.Fprintf(buf, "\t%s: %s\n", function(pc), source(lines, line))
	}
	return buf.Bytes()
}
```

## 其他打印协程的方法

debug.PrintStack可打印当前协程的栈信息，相比上面runtime.Stack和runtime.Caller，该方法只能输出到标准错误输出的fd上。为了能完整输出栈信息，还精细的做了buf的扩充重试。😅 另外，pprof也提供了栈的打印，pprof.Lookup(“goroutine”)就可以拿到。

```go
// xiaorui.cc

func PrintStack() {
	os.Stderr.Write(Stack())
}

func Stack() []byte {
	buf := make([]byte, 1024)
	for {
		n := runtime.Stack(buf, false)
		if n < len(buf) {
			return buf[:n]
		}
		buf = make([]byte, 2*len(buf))
	}
}
```

## 解决方法

修改默认值，让recovery只打印当前协程栈信息，这样就避免了加锁的各种操作了。更推荐的方法是自定义中间件来实现recovery，runtime.Caller的性能要优于runtime.Stack。

## 总结

以前文章就多次讲过，要小心golang的锁 😅。锁会造成各种的性能问题，通常表现为吞吐性能不足，指标为高延迟及阻塞。