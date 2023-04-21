# 源码分析golang http shutdown优雅退出的原理

我们知道在go 1.8.x后，golang在http里加入了shutdown方法，用来控制优雅退出。什么是优雅退出？ 简单说就是不处理新请求，但是会处理正在进行的请求，把旧请求都处理完，也就是都response之后，那么就退出。

社区里不少http graceful动态重启，平滑重启的库，大多是基于http.shutdown做的。平滑启动的原理很简单，fork子进程，继承listen fd, 老进程优雅退出。以前写过文章专门讲述在golang里如何实现平滑重启 (graceful reload)。有兴趣的朋友可以翻翻。

## http shutdown 源码分析

先来看下 http shutdown 的主方法实现逻辑。用atomic来做退出标记的状态，然后关闭各种的资源，然后一直阻塞的等待无空闲连接，每 500ms 轮询一次。

```go
// xiaorui.cc
var shutdownPollInterval = 500 * time.Millisecond

func (srv *Server) Shutdown(ctx context.Context) error {
    // 标记退出的状态
    atomic.StoreInt32(&srv.inShutdown, 1)
    srv.mu.Lock()
    // 关闭listen fd，新连接无法建立。
    lnerr := srv.closeListenersLocked()
    
    // 把server.go的done chan给close掉，通知等待的worekr退出
    srv.closeDoneChanLocked()

    // 执行回调方法，我们可以注册shutdown的回调方法
    for _, f := range srv.onShutdown {
        go f()
    }

    // 每500ms来检查下，是否没有空闲的连接了，或者监听上游传递的ctx上下文。
    ticker := time.NewTicker(shutdownPollInterval)
    defer ticker.Stop()
    for {
        if srv.closeIdleConns() {
            return lnerr
        }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
        }
    }
}
```

是否没有空闲的连接
遍历连接，当客户单的连接已空闲，则关闭连接，并在 activeConn 连接池中剔除该连接。

```go
func (s *Server) closeIdleConns() bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	quiescent := true
	for c := range s.activeConn {
		st, unixSec := c.getState()
		if st == StateNew && unixSec < time.Now().Unix()-5 {
			st = StateIdle
		}
		if st != StateIdle || unixSec == 0 {
			quiescent = false
			continue
		}
		c.rwc.Close()
		delete(s.activeConn, c)
	}
	return quiescent
}
```

关闭server.doneChan和监听的文件描述符

```go
// xiaorui.cc
// 关闭doen chan
func (s *Server) closeDoneChanLocked() {
    ch := s.getDoneChanLocked()
    select {
    case <-ch:
        // Already closed. Don't close again.
    default:
        // Safe to close here. We're the only closer, guarded
        // by s.mu.
        close(ch)
    }
}

// 关闭监听的fd
func (s *Server) closeListenersLocked() error {
    var err error
    for ln := range s.listeners {
        if cerr := (*ln).Close(); cerr != nil && err == nil {
            err = cerr
        }
        delete(s.listeners, ln)
    }
    return err
}

// 关闭连接
func (c *conn) Close() error {
    if !c.ok() {
        return syscall.EINVAL
    }
    err := c.fd.Close()
    if err != nil {
        err = &OpError{Op: "close", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return err
}
```

这么一系列的操作后，server.go的serv主监听方法也就退出了。

```go
// xiaorui.cc 
func (srv *Server) Serve(l net.Listener) error {
    ...
    for {
        rw, e := l.Accept()
        if e != nil {
            select {
             // 退出
            case <-srv.getDoneChan():
                return ErrServerClosed
            default:
            }
            ...
            return e
        }
        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)
    }
}
```

那么如何保证用户在请求完成后，再关闭连接的？

```go
// xiaorui.cc

func (s *Server) doKeepAlives() bool {
	return atomic.LoadInt32(&s.disableKeepAlives) == 0 && !s.shuttingDown()
}


// Serve a new connection.
func (c *conn) serve(ctx context.Context) {
	defer func() {
                ... xiaorui.cc ...
		if !c.hijacked() {
                        // 关闭连接，并且标记退出
			c.close()
			c.setState(c.rwc, StateClosed)
		}
	}()
        ...
	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()

	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
                // 接收请求
		w, err := c.readRequest(ctx)
		if c.r.remain != c.server.initialReadLimitSize() {
			c.setState(c.rwc, StateActive)
		}
                ...
                ...
                // 匹配路由及回调处理方法
		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.cancelCtx()
		if c.hijacked() {
			return
		}
                ...
                // 判断是否在shutdown mode, 选择退出
		if !w.conn.server.doKeepAlives() {
			return
		}
    }
    ...
```

## 总结:

总觉得golang net/http的代码写得有点乱，应该能写得更好。我也看过不少golang标准库的源代码，最让我头疼的就是net/http了。😅