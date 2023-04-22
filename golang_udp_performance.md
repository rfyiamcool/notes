# golang udp 高性能网络优化

前段时间优化了golang udp client和server的性能问题，我在这里简单描述下udp服务的优化过程。当然，udp性能本就很高，就算不优化，也轻易可以到几十万的qps，但我们想更好的优化go udp server和client。

## UDP 存在粘包半包问题 ？

 我们知道应用程序之间的网络传输会存在粘包半包的问题。该问题的由来我这里就不描述了，大家去搜吧。使用tcp会存在该问题，而udp是不存在该问题的。

为啥? tcp是无边界的，tcp是基于流传输的，tcp报头没有长度这个变量，而udp是有边界的，基于消息的，是可以解决粘包问题。udp协议里有16位来描述包的大小，16位决定他的数字最大数字是65536，除去udp头和ip头的大小，最大的包差不多是65507byte。

但根据我们的测试，udp并没有完美的解决应用层粘包半包的问题。如果你的go udp server的读缓冲是1024，那么client发送的数据不能超过 server read buf定义的1024byte，不然还是要处理半包了。 如果发送的数据小于1024 byte，倒是不会出现粘包的问题。

```go
// xiaorui.cc
buf := make([]byte, 1024)
for {
    n, _ := ServerConn.Read(buf[0:])
    if string(buf[0:n]) != s {
        panic(...)
...
```

在linux下借助strace发现syscall read fd的时候，就最大只获取1024个字节。这个1024就是上面配置的读缓冲大小。

```
// xiaorui.cc
[pid 25939] futex(0x56db90, FUTEX_WAKE, 1) = 1
[pid 25939] read(3, "Introduction... 隐藏... overview of IPython'", 1024) = 1024
[pid 25939] epoll_ctl(4, EPOLL_CTL_DEL, 3, {0, {u32=0, u64=0}}) = 0
[pid 25939] close(3 
[pid 25940] <... restart_syscall resumed> ) = 0
[pid 25939] <... close resumed> )       = 0
[pid 25940] clock_gettime(CLOCK_MONOTONIC, {19280781, 509925143}) = 0
[pid 25939] pselect6(0, NULL, NULL, NULL, {0, 1000000}, 0 
[pid 25940] pselect6(0, NULL, NULL, NULL, {0, 20000}, 0) = 0 (Timeout)
[pid 25940] clock_gettime(CLOCK_MONOTONIC, {19280781, 510266460}) = 0
[pid 25940] futex(0x56db90, FUTEX_WAIT, 0, {60, 0} 
```

下面是golang里socket fd read的源码，可以看到你传入多大的byte数组，他就syscall read多大的数据。

```go
// xiaorui.cc
func read(fd int, p []byte) (n int, err error) {
    var _p0 unsafe.Pointer
    if len(p) > 0 {
        _p0 = unsafe.Pointer(&p[0])
    } else {
        _p0 = unsafe.Pointer(&_zero)
    }
    r0, _, e1 := Syscall(SYS_READ, uintptr(fd), uintptr(_p0), uintptr(len(p)))
    n = int(r0)
    if e1 != 0 {
        err = errnoErr(e1)
    }
    return
}
```

http2为毛比http1的协议解析更快，是因为http2实现了header的hpack编码协议。thrift为啥比grpc快？单单对比协议结构体来说，thrift和protobuf的性能半斤八两，但对比网络应用层协议来说，thrift要更快。因为grpc是在http2上跑的，grpc server不仅要解析http2 header，还要解析http2 body，这个body就是protobuf数据。

所以说，高效的应用层协议也是高性能服务的重要的一个标准。我们先前使用的是自定义的TLV编码，t是类型，l是length，v是数据。一般解决网络协议上的数据完整性差不多是这个思路。当然，我也是这么搞得。

## 如何优化udp应用协议上的开销?

上面已经说了，udp在合理的size情况下是不需要依赖应用层协议解析包问题。那么我们只需要在client端控制send包的大小，server端控制接收大小，就可以节省应用层协议带来的性能高效。😁 别小看应用层协议的cpu消耗 ！！！

## 解决golang udp的锁竞争问题

在udp压力测试的时候，会发现client和server都跑不满cpu的情况。开始以为是golang udp server的问题，去掉所有相关的业务逻辑，只是单纯的做atomic计数，还是跑不满cpu。 通过go tool pprof的函数调用图以及火焰图，看不出问题所在。尝试使用iperf进行udp压测，golang udp server的压力直接干到了满负载。可以说是压力源不足。

![http://xiaorui.cc/wp-content/uploads/2019/01/4.pic_.jpg](http://xiaorui.cc/wp-content/uploads/2019/01/4.pic_.jpg)
![http://xiaorui.cc/wp-content/uploads/2019/01/2.pic_.jpg](http://xiaorui.cc/wp-content/uploads/2019/01/2.pic_.jpg)

 那么udp性能上不去的问题看似明显了，应该是golang udp client的问题了。我尝试在go udp client里增加了多协程写入，10个goroutine，100个goroutine，500个goroutine，都没有好的明显的提升效果，而且性能抖动很明显。 😅

进一步排查问题，通过lsof分析client进程的描述符列表，client连接udp server只有一个连接。也就是说，500个协程共用一个连接。 接着使用strace做syscall系统调用统计，发现futex和pselect6系统调用特别多，这一看就是存在过大的锁竞争。翻看golang net 源代码，果然发现golang在往socket fd写入时，会存在写锁竞争。

![http://xiaorui.cc/wp-content/uploads/2019/01/3.pic_meitu_1.jpg](http://xiaorui.cc/wp-content/uploads/2019/01/3.pic_meitu_1.jpg)

```go
// xiaorui.cc

// Write implements io.Writer.
func (fd *FD) Write(p []byte) (int, error) {
	if err := fd.writeLock(); err != nil {
		return 0, err
	}
	defer fd.writeUnlock()
	if err := fd.pd.prepareWrite(fd.isFile); err != nil {
		return 0, err
	}
}
```

## 怎么优化锁竞争 ?

实例化多个udp连接到一个数组池子里。。。在客户端代码里随机使用udp连接。这样就能减少锁的竞争了。

## 总结:

udp性能调优的过程就是这样子了。简单说就两个点，一个是消除应用层协议带来的性能消耗，再一个是golang socket写锁带来的竞争。 当我们一些性能问题时，多使用perf、strace功能，再配合golang pprof 分析火焰图来分析问题。实在不行，直接干golang源码。