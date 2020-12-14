## golang http client连接池不复用的问题

### 摘要

当httpclient返回值为不为空，只读取response header，但不读body内容就response.Body.Close()，那么连接会被主动关闭，得不到复用。

**测试代码如下:**

```go
// xiaorui.cc

func HttpGet() {
	for {
		fmt.Println("new")
		resp, err := http.Get("http://www.baidu.com")
		if err != nil {
			fmt.Println(err)
			continue
		}

		if resp.StatusCode == http.StatusOK {
			continue
		}

		resp.Body.Close()
		fmt.Println("go num", runtime.NumGoroutine())
	}
}
```

> 正如大家所想，除了HEAD Method外，很少会有只读取header的需求吧。

话说，golang httpclient需要注意的地方着实不少。

- 如没有response.Body.Close()，有些小场景造成persistConn的writeLoop泄露。
- 如header和body都不管，那么会造成泄露的连接干满连接池，后面的请求只能是`短连接`。

### 上下文

由于某几个业务系统会疯狂调用各区域不同的k8s集群，为减少跨机房带来的时延、新老k8s集群api兼容、减少k8s api-server的负载，故而开发了k8scache服务。在部署运行后开始对该服务进行监控，发现metrics呈现的QPS跟连接数不成正比，qps为1500，连接数为10个。开始以为触发idle timeout被回收，但通过历史监控图分析到连接依然很少。😅

按照对k8scache调用方的理解，他们经常粗暴的开启不少协程来对k8scache进行访问。已知默认的golang httpclient transport对连接数是有默认限制的，连接池总大小为100，每个host连接数为2。当并发对某url进行请求时，当无法归还连接池，也就是超过连接池大小的连接会被主动clsoe()。所以，我司的golang脚手架中会对默认的httpclient创建高配的transport，不太可能出现连接池爆满被close的问题。

如果真的是连接池爆了?  谁主动挑起关闭，谁就有tcp time-wait状态，但通过netstat命令只发现少量跟k8scache相关的time-wait。

### 排查问题

> 已知问题,  为隐藏敏感信息，索性使用简单的场景设立问题的case 

**tcpdump抓包分析问题?**

包信息如下，通过最后一行可以确认是由客户端主动触发 `RST连接重置` 。触发RST的场景有很多，但常见的有tw_bucket满了、tcp连接队列爆满且开启tcp_abort_on_overflow、配置so_linger、读缓冲区还有数据就给close。

通过linux监控和内核日志可以确认不是内核配置的问题，配置so_linger更不可能。😅 大概率就一个可能，关闭未清空读缓冲区的连接。

```bash
22:11:01.790573 IP (tos 0x0, ttl 64, id 29826, offset 0, flags [DF], proto TCP (6), length 60)
    host-46.54550 > 110.242.68.3.http: Flags [S], cksum 0x5f62 (incorrect -> 0xb894), seq 1633933317, win 29200, options [mss 1460,sackOK,TS val 47230087 ecr 0,nop,wscale 7], length 0
22:11:01.801715 IP (tos 0x0, ttl 43, id 0, offset 0, flags [DF], proto TCP (6), length 52)
    110.242.68.3.http > host-46.54550: Flags [S.], cksum 0x00a0 (correct), seq 1871454056, ack 1633933318, win 29040, options [mss 1452,nop,nop,sackOK,nop,wscale 7], length 0
22:11:01.801757 IP (tos 0x0, ttl 64, id 29827, offset 0, flags [DF], proto TCP (6), length 40)
    host-46.54550 > 110.242.68.3.http: Flags [.], cksum 0x5f4e (incorrect -> 0xb1f5), seq 1, ack 1, win 229, length 0
22:11:01.801937 IP (tos 0x0, ttl 64, id 29828, offset 0, flags [DF], proto TCP (6), length 134)
    host-46.54550 > 110.242.68.3.http: Flags [P.], cksum 0x5fac (incorrect -> 0xb4d6), seq 1:95, ack 1, win 229, length 94: HTTP, length: 94
	GET / HTTP/1.1
	Host: www.baidu.com
	User-Agent: Go-http-client/1.1

22:11:01.814122 IP (tos 0x0, ttl 43, id 657, offset 0, flags [DF], proto TCP (6), length 40)
    110.242.68.3.http > host-46.54550: Flags [.], cksum 0xb199 (correct), seq 1, ack 95, win 227, length 0
22:11:01.815179 IP (tos 0x0, ttl 43, id 658, offset 0, flags [DF], proto TCP (6), length 4136)
    110.242.68.3.http > host-46.54550: Flags [P.], cksum 0x6f4e (incorrect -> 0x0e70), seq 1:4097, ack 95, win 227, length 4096: HTTP, length: 4096
	HTTP/1.1 200 OK
	Bdpagetype: 1
	Bdqid: 0x8b3b62c400142f77
	Cache-Control: private
	Connection: keep-alive
	Content-Encoding: gzip
	Content-Type: text/html;charset=utf-8
	Date: Wed, 09 Dec 2020 14:11:01 GMT
  ...
22:11:01.815214 IP (tos 0x0, ttl 64, id 29829, offset 0, flags [DF], proto TCP (6), length 40)
    host-46.54550 > 110.242.68.3.http: Flags [.], cksum 0x5f4e (incorrect -> 0xa157), seq 95, ack 4097, win 293, length 0
22:11:01.815222 IP (tos 0x0, ttl 43, id 661, offset 0, flags [DF], proto TCP (6), length 4136)
    110.242.68.3.http > host-46.54550: Flags [P.], cksum 0x6f4e (incorrect -> 0x07fa), seq 4097:8193, ack 95, win 227, length 4096: HTTP
22:11:01.815236 IP (tos 0x0, ttl 64, id 29830, offset 0, flags [DF], proto TCP (6), length 40)
    host-46.54550 > 110.242.68.3.http: Flags [.], cksum 0x5f4e (incorrect -> 0x9117), seq 95, ack 8193, win 357, length 0
22:11:01.815243 IP (tos 0x0, ttl 43, id 664, offset 0, flags [DF], proto TCP (6), length 5848)
    ...
    host-46.54550 > 110.242.68.3.http: Flags [.], cksum 0x5f4e (incorrect -> 0x51ba), seq 95, ack 24165, win 606, length 0
22:11:01.815369 IP (tos 0x0, ttl 64, id 29834, offset 0, flags [DF], proto TCP (6), length 40)
    host-46.54550 > 110.242.68.3.http: Flags [R.], cksum 0x5f4e (incorrect -> 0x51b6), seq 95, ack 24165, win 606, length 0
```

通过lsof找到进程关联的TCP连接，然后使用ss或netstat查看读写缓冲区。信息如下，recv-q读缓冲区确实是存在数据。这个缓冲区字节一直未读，直到连接关闭引发了rst。

```bash
$ lsof -p 54330
COMMAND   PID USER   FD      TYPE    DEVICE SIZE/OFF       NODE NAME
...
aaa     54330 root    1u      CHR     136,0      0t0          3 /dev/pts/0
aaa     54330 root    2u      CHR     136,0      0t0          3 /dev/pts/0
aaa     54330 root    3u  a_inode      0,10        0       8838 [eventpoll]
aaa     54330 root    4r     FIFO       0,9      0t0  223586913 pipe
aaa     54330 root    5w     FIFO       0,9      0t0  223586913 pipe
aaa     54330 root    6u     IPv4 223596521      0t0        TCP host-46:60626->110.242.68.3:http (ESTABLISHED)

$ ss -an|egrep "68.3:80"
State      Recv-Q      Send-Q       Local Address:Port        Peer Address:Port 
ESTAB      72480       0            172.16.0.46:60626         110.242.68.3:80	
```

**strace跟踪系统调用**

通过系统调用可分析出，貌似只是读取了header部分了，还未读到body的数据。

```bash
[pid  8311] connect(6, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("110.242.68.3")}, 16 <unfinished ...>
[pid 195519] epoll_pwait(3,  <unfinished ...>
[pid  8311] <... connect resumed>)      = -1 EINPROGRESS (操作现在正在进行)
[pid  8311] epoll_ctl(3, EPOLL_CTL_ADD, 6, {EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, {u32=2350546712, u64=140370471714584}} <unfinished ...>
[pid 195519] getsockopt(6, SOL_SOCKET, SO_ERROR,  <unfinished ...>
[pid 192592] nanosleep({tv_sec=0, tv_nsec=20000},  <unfinished ...>
[pid 195519] getpeername(6, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("110.242.68.3")}, [112->16]) = 0
[pid 195519] getsockname(6,  <unfinished ...>
[pid 195519] <... getsockname resumed>{sa_family=AF_INET, sin_port=htons(47746), sin_addr=inet_addr("172.16.0.46")}, [112->16]) = 0
[pid 195519] setsockopt(6, SOL_TCP, TCP_KEEPINTVL, [15], 4) = 0
[pid 195519] setsockopt(6, SOL_TCP, TCP_KEEPIDLE, [15], 4 <unfinished ...>
[pid  8311] write(6, "GET / HTTP/1.1\r\nHost: www.baidu.com\r\nUser-Agent: Go-http-client/1.1\r\nAccept-Encoding: gzip\r\n\r\n", 94 <unfinished ...>
[pid 192595] read(6,  <unfinished ...>
[pid 192595] <... read resumed>"HTTP/1.1 200 OK\r\nBdpagetype: 1\r\nBdqid: 0xc43c9f460008101b\r\nCache-Control: private\r\nConnection: keep-alive\r\nContent-Encoding: gzip\r\nContent-Type: text/html;charset=utf-8\r\nDate: Wed, 09 Dec 2020 13:46:30 GMT\r\nExpires: Wed, 09 Dec 2020 13:45:33 GMT\r\nP3p: CP=\" OTI DSP COR IVA OUR IND COM \"\r\nP3p: CP=\" OTI DSP COR IVA OUR IND COM \"\r\nServer: BWS/1.1\r\nSet-Cookie: BAIDUID=996EE645C83622DF7343923BF96EA1A1:FG=1; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com\r\nSet-Cookie: BIDUPSID=99"..., 4096) = 4096
[pid 192595] close(6 <unfinished ...>
```

**逻辑代码**

那么到这里，可以大概猜测问题所在，找到业务方涉及到httpclient的逻辑代码。伪代码如下，跟上面的结论一样，只是读取了header，但并未读取完response body数据。

> 还以为是特殊的场景，结果是使用不当，把请求投递过去后只判断http code？ 真正的业务code是在body里的。😅

```go
urls := []string{...}
for _, url := range urls {
		resp, err := http.Post(url, ...)
		if err != nil {
			// ...
		}
		if resp.StatusCode == http.StatusOK {
			continue
		}

		// handle redis cache
		// handle mongodb
		// handle rocketmq
		// ...

		resp.Body.Close()
}
```

**如何解决**

不细说了，把header length长度的数据读完就可以了。

### 分析问题

先不管别人使用不当，再分析下为何出现短连接，连接不能复用的问题。

为什么不读取body就出问题？其实http.Response字段描述中已经有说明了。当Body未读完时，连接可能不能复用。

```go
	// The http Client and Transport guarantee that Body is always
	// non-nil, even on responses without a body or responses with
	// a zero-length body. It is the caller's responsibility to
	// close Body. The default HTTP client's Transport may not
	// reuse HTTP/1.x "keep-alive" TCP connections if the Body is
	// not read to completion and closed.
	//
	// The Body is automatically dechunked if the server replied
	// with a "chunked" Transfer-Encoding.
	//
	// As of Go 1.12, the Body will also implement io.Writer
	// on a successful "101 Switching Protocols" response,
	// as used by WebSockets and HTTP/2's "h2c" mode.
	Body io.ReadCloser
```

众所周知，golang httpclient要注意response Body关闭问题，但上面的case确实有关了body，只是非常规地没去读取reponse body数据。这样会造成连接异常关闭，继而引起连接池不能复用。

一般http协议解释器是要先解析header，再解析body，结合当前的问题开始是这么推测的，连接的readLoop收到一个新请求，然后尝试解析header后，返回给调用方等待读取body，但调用方没去读取，而选择了直接关闭body。那么后面当一个新请求被transport roundTrip再调度请求时，readLoop的header读取和解析会失败，因为他的读缓冲区里有前面未读的数据，必然无法解析header。按照常见的网络编程原则，协议解析失败，直接关闭连接。

想是这么想的，但还是看了golang net/http的代码，结果不是这样的。😅

### 分析源码

httpclient每个连接会创建读写协程两个协程，分别使用reqch和writech来跟roundTrip通信。上层使用的response.Body其实是经过多次封装的，一次层封装的body是直接跟net.conn进行交互读取，二次封装的body则是加强了close和eof处理的bodyEOFSignal。

当未读取body就进行close时，会触发earlyCloseFn()回调，看earlyCloseFn的函数定义，在close未见io.EOF时才调用。自定义的earlyCloseFn方法会给readLoop监听的waitForBodyRead传入false,  这样引发alive为false不能继续循环的接收新请求，只能是退出调用注册过的defer方法，关闭连接和清理连接池。

```go
// xiaorui.cc

func (pc *persistConn) readLoop() {
	closeErr := errReadLoopExiting // default value, if not changed below
	defer func() {
		pc.close(closeErr)      // 关闭连接
		pc.t.removeIdleConn(pc) // 从连接池中删除
	}()

  ...

	alive := true
	for alive {
	    ...

		rc := <-pc.reqch  // 从管道中拿到请求，roundTrip对该管道进行输入
		trace := httptrace.ContextClientTrace(rc.req.Context())

		var resp *Response
		if err == nil {
			resp, err = pc.readResponse(rc, trace) // 更多的是解析header
		} else {
			err = transportReadFromServerError{err}
			closeErr = err
		}
    ...

		waitForBodyRead := make(chan bool, 2)
		body := &bodyEOFSignal{
			body: resp.Body,
			// 提前关闭 !!! 输出false
			earlyCloseFn: func() error {
				waitForBodyRead <- false
				...
			},
			// 正常收尾 !!!
			fn: func(err error) error {
				isEOF := err == io.EOF
				waitForBodyRead <- isEOF
				...
			},
		}

		resp.Body = body

		select {
		case rc.ch <- responseAndError{res: resp}:
		case <-rc.callerGone:
			return
		}

		select {
		case bodyEOF := <-waitForBodyRead:
			replaced := pc.t.replaceReqCanceler(rc.cancelKey, nil) // before pc might return to idle pool
			// alive为false, 不能继续continue
			alive = alive &&
				bodyEOF &&
				!pc.sawEOF &&
				pc.wroteRequest() &&
				replaced && tryPutIdleConn(trace)
				...
		case <-rc.req.Cancel:
			alive = false
			pc.t.CancelRequest(rc.req)
		case <-rc.req.Context().Done():
			alive = false
			pc.t.cancelRequest(rc.cancelKey, rc.req.Context().Err())
		case <-pc.closech:
			alive = false
		}
	}
}
```

bodyEOFSignal的Close()

```
// xiaorui.cc

func (es *bodyEOFSignal) Close() error {
	es.mu.Lock()
	defer es.mu.Unlock()
	if es.closed {
		return nil
	}
	es.closed = true
	if es.earlyCloseFn != nil && es.rerr != io.EOF {
		return es.earlyCloseFn()
	}
	err := es.body.Close()
	return es.condfn(err)
}

```

最终会调用persistConn的close(), 连接关闭并关闭closech。

```go
// xiaorui.cc

func (pc *persistConn) close(err error) {
	pc.mu.Lock()
	defer pc.mu.Unlock()
	pc.closeLocked(err)
}

func (pc *persistConn) closeLocked(err error) {
	if err == nil {
		panic("nil error")
	}
	pc.broken = true
	if pc.closed == nil {
		pc.closed = err
		pc.t.decConnsPerHost(pc.cacheKey)
		if pc.alt == nil {
			if err != errCallerOwnsConn {
				pc.conn.Close() // 关闭连接
			}
			close(pc.closech) // 通知读写协程
		}
	}
}
```

### 总之

同事的httpclient使用方法有些奇怪，除了head method之外，还真想不到有不读取body的请求。所以，大家知道httpclient有这么一回事就行了。

另外，一直觉得net/http的代码太绕，没看过一些介绍直接看代码很容易陷进去，有时间专门讲讲http client的实现。