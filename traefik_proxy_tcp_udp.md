## traefik 设计实现之 tcp 和 udp 代理

traefik 处理 http 的代理外，还支持 tcp, udp 的代理转发. 这里通过 traefik 源码来分析下 tcp 和 udp 代理的设计实现.

### TCP 代理

先 services 创建一个 TCP 负载均衡器对象，传入 tcp.NewProxy 对象，后面有请求到来时，调用 Proxy 的 ServeTCP 接口来处理请求.

#### TCP 负载均衡器

`BuildTCP()` 遍历后端地址列表, 生成一个个的 proxy 代理对象并放到一个均衡器里，该均衡器实现了 `tcp.Hanlder` 接口. `tcp.Hanlder` 接口就一个 `ServeTcp()` 方法. 这里的 ServeTcp() 其实就是从负载均衡器里按照算法获取 tcp.Proxy 对象， 然后再调用 `tcp.Proxy` 对象的 `ServeTcp` 对象.

`pkg/server/service/tcp/service.go`

```go
type Manager struct {
	configs map[string]*runtime.TCPServiceInfo
}

// NewManager creates a new manager.
func NewManager(conf *runtime.Configuration) *Manager {
	return &Manager{
		configs: conf.TCPServices,
	}
}

func (m *Manager) BuildTCP(rootCtx context.Context, serviceName string) (tcp.Handler, error) {
	...
	switch {
	case conf.LoadBalancer != nil:
		loadBalancer := tcp.NewWRRLoadBalancer()

		if conf.LoadBalancer.TerminationDelay == nil {
			conf.LoadBalancer.TerminationDelay = 100
		}
		duration := time.Duration(*conf.LoadBalancer.TerminationDelay) * time.Millisecond

		for name, server := range shuffle(conf.LoadBalancer.Servers, m.rand) {
			if _, _, err := net.SplitHostPort(server.Address); err != nil {
			}

			handler, err := tcp.NewProxy(server.Address, duration, conf.LoadBalancer.ProxyProtocol)
			loadBalancer.AddServer(handler)
			logger.WithField(log.ServerName, name).Debugf("Creating TCP server %d at %s", name, server.Address)
		}
		return loadBalancer, nil
	case conf.Weighted != nil:
		loadBalancer := tcp.NewWRRLoadBalancer()

		for _, service := range shuffle(conf.Weighted.Services, m.rand) {
			handler, err := m.BuildTCP(rootCtx, service.Name)
			loadBalancer.AddWeightServer(handler, service.Weight)
		}
		return loadBalancer, nil
	default:
	}
}
```

#### TCP 代理

创建一个 `tcp.Proxy` 代理对象，该对象实现 `ServeTCP` 接口. 这里 `ServeTcp` 的实现是这样， 当传递一个客户端连接对象时, 开启两个协程，一个是流入方向的 `io.copy` 复制，一个是流出方向的 `io.copy` , 然后再加上一个等待这两 io 协程返回的 goroutine, 这样一个 tcp 代理共需要 3 个协程.

`pkg/tcp/proxy.go`

```go
func NewProxy(address string, terminationDelay time.Duration, proxyProtocol *dynamic.ProxyProtocol) (*Proxy, error) {
	return &Proxy{
		address:          address,
		tcpAddr:          tcpAddr,
		terminationDelay: terminationDelay,
		proxyProtocol:    proxyProtocol,
	}, nil
}

// ServeTCP forwards the connection to a service.
func (p *Proxy) ServeTCP(conn WriteCloser) {
	defer conn.Close()
	connBackend, err := p.dialBackend()

	errChan := make(chan error)
	if p.proxyProtocol != nil && p.proxyProtocol.Version > 0 && p.proxyProtocol.Version < 3 {
		header := proxyproto.HeaderProxyFromAddrs(byte(p.proxyProtocol.Version), conn.RemoteAddr(), conn.LocalAddr())
		if _, err := header.WriteTo(connBackend); err != nil {
			return
		}
	}

	go p.connCopy(conn, connBackend, errChan)
	go p.connCopy(connBackend, conn, errChan)

	err = <-errChan
	if err != nil {
	}

	<-errChan
}

func (p Proxy) dialBackend() (*net.TCPConn, error) {
	conn, err := net.Dial("tcp", p.address)
	if err != nil {
		return nil, err
	}

	return conn.(*net.TCPConn), nil
}

func (p Proxy) connCopy(dst, src WriteCloser, errCh chan error) {
	_, err := io.Copy(dst, src)
	errCh <- err

	...
	if p.terminationDelay >= 0 {
		err := dst.SetReadDeadline(time.Now().Add(p.terminationDelay))
		if err != nil {
			log.WithoutContext().Debugf("Error while setting deadline: %v", err)
		}
	}
}
```

### UDP proxy

`NewUDPEntryPoint()` 创建一个 udpPoint 对象，然后 `Start()` 不断的 accept 获取新的 conn 连接对象， 然后调用 udp.Proxy 的 ServeUDP 方法，该方法的逻辑跟 tcp.Proxy 类似，也是开启两个协程，流入和流出的 `io.copy`.

> udp server 不存在 accept 获取文件描述符的行为, traefik udp accept 是抽象的行为. 另外写数据时直接使用 `listenFD.WriteTo([]byte, addr)` 就可以了.

```go
func (p *Proxy) ServeUDP(conn *Conn) {
	connBackend, err := net.Dial("udp", p.target)
	...

	errChan := make(chan error)
	go connCopy(conn, connBackend, errChan)
	go connCopy(connBackend, conn, errChan)

	err = <-errChan
}
```

traefik 关于 udp 没什么可以说的，源码也相对简单.

### golang io.Copy

#### 数据零拷贝

通过上面可以得知，tcp/udp 数据的拷贝采用了 `io.Copy`, 那么 `io.Copy` 性能好么？

golang 的 `io.Copy` 是个极好的实现，不仅实现 golang 内置 io 对象之间的拷贝，而且对 sendfile 和 splice 系统调用做了封装. 这两个系统调用都有零拷贝的能力，不仅可以减少数据在用户态和内核态之间的来回拷贝，而且可以减少切换次数.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212031934017.png)

`sendfile()` 开始时只支持目标为 socket 文件描述符的拷贝，但在后面的 linux 版本之后，已经可以支持任意类型的文件描述符，但是 `输入文件描述符依然只能指向文件`. 

所以，Linux 在 `2.6.17` 版本引入了一个新的系统调用 `splice()`，它在功能上和 `sendfile()` 非常相似，但是能够实现在任意类型的两个文件描述符时之间传输数据. 

虽然这俩都做了加强，可以支持更多的类型，但从社区的使用上来说，`sendfile()` 通常用在文件到 `socket` 写缓冲区之间的拷贝，而 splice 可以用在两个 `socket` 之间的拷贝.  像 haproxy 也支持 splice 做代理，但不知道为什么 nginx stream 没有使用 splice 实现代理.

下面是 strace 追踪 nginx stream 转发过程.

```bash
accept4(8, {sa_family=AF_INET, sin_port=htons(34241), sin_addr=inet_addr("127.0.0.1")}, [112->16], SOCK_NONBLOCK) = 4

// 请求客户端的文件描述符 = 4
setsockopt(4, SOL_TCP, TCP_NODELAY, [1], 4) = 0

epoll_ctl(12, EPOLL_CTL_ADD, 5, {EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, {u32=4073727168, u64=139805259211968}}) = 0

// 连接后端的文件描述符 = 5
connect(5, {sa_family=AF_INET, sin_port=htons(12001), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS (操作现在正在进行)
epoll_wait(12, [{EPOLLOUT, {u32=4073727168, u64=139805259211968}}], 512, 1000) = 1

epoll_ctl(12, EPOLL_CTL_ADD, 4, {EPOLLIN|EPOLLRDHUP|EPOLLET, {u32=4073726928, u64=139805259211728}}) = 0
epoll_wait(12, [{EPOLLIN, {u32=4073726928, u64=139805259211728}}], 512, 3000) = 1

recvfrom(4, "xiaorui.cc\n", 16384, 0, NULL, NULL) = 11
writev(5, [{iov_base="xiaorui.cc\n", iov_len=11}], 1) = 11
epoll_wait(12, [], 512, 3000)           = 0

close(5)                                = 0
close(4)                                = 0
```

另外, envoy 也没有使用纯内核态的 splice 实现 tcp 代理，而是使用用户态的数据拷贝. 😅 搞不懂.

####  io.copy 的 splice 代码分析

下面是对 io.Copy 代码的解析，首先需要判断目标的 io 对象是否实现了 ReaderFrom 接口， 然后调用 ReadFrom 方法，该方法里会判断 IO 对象的文件描述符类型.

如果目标类型是 socket 或者 unix socket domain ，则使用 splice. 

如果源描述符类型是 os.File 文件，则使用 sendfile 调用.

```
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	...
}
```

readFrom 实现

```go
func (c *TCPConn) readFrom(r io.Reader) (int64, error) {
 if n, err, handled := splice(c.fd, r); handled {
  return n, err
 }
 if n, err, handled := sendFile(c.fd, r); handled {
  return n, err
 }
 return genericReadFrom(c, r)
}
```

splice 实现

```go
func splice(c *netFD, r io.Reader) (written int64, err error, handled bool) {
	var remain int64 = 1 << 62 // by default, copy until EOF
	lr, ok := r.(*io.LimitedReader)
	if ok {
		remain, r = lr.N, lr.R
		if remain <= 0 {
			return 0, nil, true
		}
	}

	var s *netFD
	if tc, ok := r.(*TCPConn); ok {
		s = tc.fd
	} else if uc, ok := r.(*UnixConn); ok {
		if uc.fd.net != "unix" {
			return 0, nil, false
		}
		s = uc.fd
	} else {
		return 0, nil, false
	}

	written, handled, sc, err := poll.Splice(&c.pfd, &s.pfd, remain)
	if lr != nil {
		lr.N -= written
	}
	return written, wrapSyscallError(sc, err), handled
}
```

splice 底层需要 pipe 管道来支持，用管道来衔接连接源和目标文件描述符.

```
func Splice(dst, src *FD, remain int64) (written int64, handled bool, sc string, err error) {
	p, sc, err := getPipe()
	if err != nil {
		return 0, false, sc, err
	}
	defer putPipe(p)
	var inPipe, n int
	for err == nil && remain > 0 {
		inPipe, err = spliceDrain(p.wfd, src, max)
		handled = handled || (err != syscall.EINVAL)
		if err != nil || inPipe == 0 {
			break
		}
		n, err = splicePump(dst, p.rfd, inPipe)
		}
	}
	if err != nil {
		return written, handled, "splice", err
	}
	return written, true, "", nil
}

func splice(out int, in int, max int, flags int) (int, error) {
	n, err := syscall.Splice(in, nil, out, nil, max, flags)
	return int(n), err
}

```

