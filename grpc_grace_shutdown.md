# 源码分析 golang grpc graceful shutdown 优雅退出

以前写过golang net/http graceful shutdown的实现原理，最近几个项目里都有用到grpc的gracefulStop方法，所以就好奇golang grpc优雅退出是如何实现的？

## http2 goaway帧

grpc的通信协议是http2，http2对于连接关闭使用goaway帧信号。goaway帧（类型= 0x7）用于启动连接关闭或发出严重错误状态信号。 goaway允许端点正常停止接受新的流，同时仍然完成对先前建立的流的处理。这可以实现管理操作，例如服务器维护，升级等。

## grpc服务端的实现:

golang grpc server提供了两个退出方法，一个是stop，一个是gracefulStop。先说下gracefulStop。首先close listen fd，这样就无法建立新的请求，然后遍历所有的当前连接发送goaway帧信号。goaway帧信号在http2用来关闭连接的。serveWG.Wait()会等待所有 handleRawConn协程的退出，在grpc server里每个新连接都会创建一个 handleRawConn协程，并且增加waitgroup的计数。

```go
// xiaorui.cc
func (s *Server) GracefulStop() {
    s.mu.Lock()
    ...

    // 关闭 listen fd，不再接收新的连接
    for lis := range s.lis {
        lis.Close()
    }

    s.lis = nil
    if !s.drain {
        for st := range s.conns {
            // 给所有的客户端发布goaway信号
            st.Drain()  
        }
        s.drain = true
    }


    // 等待所有handleRawConn协程退出，每个请求都是一个协程，通过waitgroup控制.
    s.serveWG.Wait()

    // 当还有空闲连接时，需要等待。在退出serveStreams逻辑时，会进行Broadcast唤醒。只要有一个客户端退出就会触发removeConn继而进行唤醒。
    for len(s.conns) != 0 {
        s.cv.Wait()
    }
...
```

看下drain方法的具体实现，构建goaway请求塞到controlbuf里，由grpc唯一的loopyWriter来写入报文。

```go
// xiaorui.cc

// 构建goaway请求塞入buf里，然后由统一的loopyWriter来发送报文。
func (t *http2Server) drain(code http2.ErrCode, debugData []byte) {
    t.mu.Lock()
    defer t.mu.Unlock()
    if t.drainChan != nil {
        return
    }
    t.drainChan = make(chan struct{})
    t.controlBuf.put(&goAway{code: code, debugData: debugData, headsUp: true})
}
...
```

stop方法相比gracefulStop来说，减少了goaway帧的发送，等待连接的退出。

## grpc客户端的实现

grpc客户端会new一个协程来执行reader方法，一直监听新数据的到来，当帧类型为goaway时调用handleGoAway，该方法会调用closeStream关闭当前连接的所有活动stream。对于开发者来说，只需监听grpc接口中的ctx就得到状态变更。

```go
// xiaorui.cc

// 接收各类报文
func (t *http2Client) reader() {
    ...
    for {
        t.controlBuf.throttle()
        frame, err := t.framer.fr.ReadFrame()
        switch frame := frame.(type) {
        // 接收goaway信号，回调handleGoAway方法
        case *http2.GoAwayFrame:
            t.handleGoAway(frame)
            ...
        }
    }
}

// 当前连接里的所有的活动stream进行closeStream
func (t *http2Client) handleGoAway(f *http2.GoAwayFrame) {
    ...
    for streamID, stream := range t.activeStreams {
        if streamID > id && streamID <= upperLimit {
            atomic.StoreUint32(&stream.unprocessed, 1)
            t.closeStream(stream, errStreamDrain, false, http2.ErrCodeNo, statusGoAway, nil, false)
        }
    }
    active := len(t.activeStreams)
    t.mu.Unlock()
    if active == 0 {
        t.Close()
    }
...
```

通常该连接不可用后，如客户端再次进行unary或streming请求时，grpc会按照规则来实例化新的连接，比如通过dns或者grpc balancer来地址变更。

## 抓包分析

关闭的一方会发出goaway帧。

![http://xiaorui.cc/wp-content/uploads/2020/01/123123.jpg](http://xiaorui.cc/wp-content/uploads/2020/01/123123.jpg)

## 对比net/http 和 grpc的graceful shutdown

golang的net/http在graceful实现上不会主动的关闭连接，除非是配置了强制超时退出。因为当你去主动关闭长连接时，有些低质量的客户端可能会出现异常。所以，像nginx、netty、net/http这类服务端不会主动关闭客户端的连接。

但grpc就不同了… 两端的代码本就是自动生成的，质量颇高。利用http2的goaway特性来通知关闭，规避了强制关闭引起的异常。

## 总结:

python grpc的优雅退出跟golang grpc一样的。。。虽然没看其他语言的实现，但估摸是按照一个路数 “翻译” 的。