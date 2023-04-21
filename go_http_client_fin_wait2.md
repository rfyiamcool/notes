# golang net/http超时引发大量fin-wait2

通过grafana监控面板，发现几个高频的业务缓存节点出现了大量的fin-wait2，而且fin-wait2状态持续了不短的时间。通过连接的ip地址及进行抓包数据判断出对端的业务。除此之外，频繁的去创建新连接，我们对golang net/http transport的连接池已优化过，但established已建连的连接没有得到复用。

另外，随之带来的问题是大量time-wait的出现，毕竟fin-wait2在拿到对端fin后会转变为time-wait状态。但该状态是正常的。

## 分析问题

通过业务日志发现了大量的接口超时问题，连接的地址跟netstat中fin-wait2目的地址是一致的。那么问题已经明确了，当http的请求触发超时，定时器对连接对象进行了关闭。这边都close了，那么连接自然无法复用，所以就需要创建新连接，但由于对端的API接口出现逻辑阻塞，自然就又触发了超时，continue。😅

```c
Get "http://xxxx": context deadline exceeded (Client.Timeout exceeded while awaiting headers)

Get "http://xxxx": context deadline exceeded (Client.Timeout exceeded while awaiting headers)

Get "http://xxxx": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

通过strace追踪socket的系统调用，发现golang的socket读写超时没有使用setsockopt so_sndtimeo so_revtimeo参数。

```c
[pid 34262] epoll_ctl(3, EPOLL_CTL_ADD, 6, {EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, {u32=1310076696, u64=140244877192984}}) = 0
[pid 34265] epoll_pwait(3,  <unfinished ...>
[pid 34262] <... getsockname resumed>{sa_family=AF_INET, sin_port=htons(45242), sin_addr=inet_addr("127.0.0.1")}, [112->16]) = 0
[pid 34264] epoll_pwait(3,  <unfinished ...>
[pid 34262] setsockopt(6, SOL_TCP, TCP_NODELAY, [1], 4 <unfinished ...>
[pid 34262] setsockopt(6, SOL_SOCKET, SO_KEEPALIVE, [1], 4 <unfinished ...>
[pid 34264] read(4,  <unfinished ...>
[pid 34262] setsockopt(6, SOL_TCP, TCP_KEEPINTVL, [30], 4 <unfinished ...>
...
```

## 代码分析

通过net/http源码可以看到socket的超时控制是通过定时器来实现的，在连接的roundTrip方法看到超时引发关闭连接的逻辑。由于http的语义不支持多路复用，所以为了规避超时后再回来的数据造成混乱，索性直接关闭连接。

当触发超时会主动关闭连接，这里涉及到了四次挥手，作为关闭方会发送fin，对端内核会回应ack，这时候客户端从fin-wait1到fin-wait2，而服务端在close-wait状态，等待触发close syscall系统调用。服务端什么时候触发close动作？ 需要等待net/http handler业务逻辑执行完毕。

![http://xiaorui.cc/wp-content/uploads/2020/08/Jietu20200816-151510.jpg](http://xiaorui.cc/wp-content/uploads/2020/08/Jietu20200816-151510.jpg)

```go
// xiaorui.cc

var errTimeout error = &httpError{err: "net/http: timeout awaiting response headers", timeout: true}

func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
    for {
        testHookWaitResLoop()
        select {
        case err := <-writeErrCh:
            if debugRoundTrip {
                req.logf("writeErrCh resv: %T/%#v", err, err)
            }
            if err != nil {
                pc.close(fmt.Errorf("write error: %v", err))
                return nil, pc.mapRoundTripError(req, startBytesWritten, err)
            }
            if d := pc.t.ResponseHeaderTimeout; d > 0 {
                if debugRoundTrip {
                    req.logf("starting timer for %v", d)
                }
                timer := time.NewTimer(d)
                defer timer.Stop() // prevent leaks
                respHeaderTimer = timer.C
            }
        case <-pc.closech:
            ...
        case <-respHeaderTimer:
            if debugRoundTrip {
                req.logf("timeout waiting for response headers.")
            }
            pc.close(errTimeout)
            return nil, errTimeout
```

## 如何解决？

要么加大客户端的超时时间，要么优化对端的获取数据的逻辑，总之减少超时的触发。 这个问题跟golang是没有问题，换成openresyt和python同样有这个问题。