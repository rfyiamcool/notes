# go http server感应连接中断及超时控制

长时间访问接口无返回，是让人恼火的，我一直都建议同事们在客户端及服务端加入超时快控制，有异常要及时返回，不要做盲目的等待。

如果你的http业务逻辑中含有超时逻辑，比如访问第三方的api、访问数据库等，那么golang里如何控制超时？

golang http server的启动参数里是有readTimeout和writeTimeout参数，然而这两个超时的阈值只控制网络io读写时间。拿writeTimeout 5s来说，当往socket fd写数据，超过5s还未写完，那么则返回error。

除了超时之外，业务handler又如何感应客户端的连接的异常。

## 感应客户端连接关闭

你的http handler如何感应到连接异常？ 在业务handler里可以加入context监听，这样当客户端连接关闭时，http server会调用ctx对应的cancel。在handler里可以访问外部的http及数据库，现在的网络库基本都支持传递ctx并监听ctx，这样只要源头上做了ctx cancel，后面的方法调用都会及时的退出。当然前提是涉及到的方法会监听ctx，不然是白扯。

**测试代码如下:**

```go
// xiaorui.cc

package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"time"
)

func handler(w http.ResponseWriter, r *http.Request) {
	result := testCall(r.Context())
	io.WriteString(w, result+"\n")
}

func testCall(ctx context.Context) string {
	var ts = time.Duration(5) * time.Second
	select {
	case <-ctx.Done():
		log.Printf("to cancel")
		return "ctx done"

	case <-time.After(ts):
		log.Printf("timeout %v", ts)
		return "hello world"
	}
}

func main() {
	srv := http.Server{
		Addr:    ":8888",
		Handler: http.HandlerFunc(handler),
		// Handler:      http.TimeoutHandler(http.HandlerFunc(handler), 2*time.Second, "Timeout!\n"),
	}

	if err := srv.ListenAndServe(); err != nil {
		fmt.Printf("Server failed: %s\n", err)
	}
}
```

## http timeHandler 超时控制

http.TimeoutHandler可理解为http控制超时的装饰器，对一个业务handler方法实现超时控制, 如超过2s还未return，那么会cancel context，且返回http code 503。

```go
http.TimeoutHandler(http.HandlerFunc(handler), 2*time.Second, "Timeout!\n") ...
```

可以跑个例子，timeoutHandler设置2s，curl访问时会受到http code = 503及response body = Timeout !

```go
HTTP/1.1 503 Service Unavailable
Date: Sat, 31 Oct 2020 09:21:01 GMT
Content-Length: 9
Content-Type: text/plain; charset=utf-8

Timeout!
```

### 官方 godoc 文档

```go
func TimeoutHandler ¶
func TimeoutHandler(h Handler, dt time.Duration, msg string) Handler
TimeoutHandler returns a Handler that runs h with the given time limit.

The new Handler calls h.ServeHTTP to handle each request, but if a call runs for longer than its time limit, the handler responds with a 503 Service Unavailable error and the given message in its body. (If msg is empty, a suitable default message will be sent.) After such a timeout, writes by h to its ResponseWriter will return ErrHandlerTimeout.

TimeoutHandler supports the Pusher interface but does not support the Hijacker or Flusher interfaces.
```

### TimeoutHandler 源码实现

timeoutHandler继承了http request的ctx，又通过该ctx生成可超时的子ctx，然后给具体的业务handler传递该子ctx。timeoutHandler里会重新开一个协程来执行传入的handler，如果handler没有监听ctx又触发了超时，那么可能存在协程泄露的危险。

这里为了规避同时触发超时写返回及业务写返回，所以抽象了timeoutWriter结构体，在writer中加入对header和body的锁安全控制。通过 wroteHeader 字段来标记是否已经写过了。比如 timeoutHandler先加锁再操作，后面handler继续操作时，由于已写会跳过。

```go
// xiaorui.cc

// TimeoutHandler returns a Handler that runs h with the given time limit.
func TimeoutHandler(h Handler, dt time.Duration, msg string) Handler {
	return &timeoutHandler{
		handler: h,
		body:    msg,
		dt:      dt,
	}
}

var ErrHandlerTimeout = errors.New("http: Handler timeout")

type timeoutHandler struct {
	handler Handler
	body    string
	dt      time.Duration

	// When set, no context will be created and this context will
	// be used instead.
	testContext context.Context
}

func (h *timeoutHandler) errorBody() string {
	if h.body != "" {
		return h.body
	}
	return "<html><head><title>Timeout</title></head><body><h1>Timeout</h1></body></html>"
}

func (h *timeoutHandler) ServeHTTP(w ResponseWriter, r *Request) {
	ctx := h.testContext
	if ctx == nil {
		var cancelCtx context.CancelFunc
		ctx, cancelCtx = context.WithTimeout(r.Context(), h.dt)
		defer cancelCtx()
	}
	r = r.WithContext(ctx)
	done := make(chan struct{})
	tw := &timeoutWriter{
		w:   w,
		h:   make(Header),
		req: r,
	}
	panicChan := make(chan interface{}, 1)
	go func() {
		defer func() {
			if p := recover(); p != nil {
				panicChan <- p
			}
		}()
		h.handler.ServeHTTP(tw, r)
		close(done)
	}()
	select {
	case p := <-panicChan:
		panic(p)
	case <-done:
		tw.mu.Lock()
		defer tw.mu.Unlock()
		dst := w.Header()
		for k, vv := range tw.h {
			dst[k] = vv
		}
		if !tw.wroteHeader {
			tw.code = StatusOK
		}
		w.WriteHeader(tw.code)
		w.Write(tw.wbuf.Bytes())
	case <-ctx.Done():
		tw.mu.Lock()
		defer tw.mu.Unlock()
		w.WriteHeader(StatusServiceUnavailable)
		io.WriteString(w, h.errorBody())
		tw.timedOut = true
	}
}

type timeoutWriter struct {
	w    ResponseWriter
	h    Header
	wbuf bytes.Buffer
	req  *Request

	mu          sync.Mutex
	timedOut    bool
	wroteHeader bool
	code        int
}

var _ Pusher = (*timeoutWriter)(nil)

// Push implements the Pusher interface.
func (tw *timeoutWriter) Push(target string, opts *PushOptions) error {
	if pusher, ok := tw.w.(Pusher); ok {
		return pusher.Push(target, opts)
	}
	return ErrNotSupported
}

func (tw *timeoutWriter) Header() Header { return tw.h }

func (tw *timeoutWriter) Write(p []byte) (int, error) {
	tw.mu.Lock()
	defer tw.mu.Unlock()
	if tw.timedOut {
		return 0, ErrHandlerTimeout
	}
	if !tw.wroteHeader {
		tw.writeHeaderLocked(StatusOK)
	}
	return tw.wbuf.Write(p)
}

func (tw *timeoutWriter) writeHeaderLocked(code int) {
	checkWriteHeaderCode(code)

	switch {
	case tw.timedOut:
		return
	case tw.wroteHeader:
		if tw.req != nil {
			caller := relevantCaller()
			logf(tw.req, "http: superfluous response.WriteHeader call from %s (%s:%d)", caller.Function, path.Base(caller.File), caller.Line)
		}
	default:
		tw.wroteHeader = true
		tw.code = code
	}
}

func (tw *timeoutWriter) WriteHeader(code int) {
	tw.mu.Lock()
	defer tw.mu.Unlock()
	tw.writeHeaderLocked(code)
}
```

