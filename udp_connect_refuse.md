## 让人迷糊的 socket udp 连接问题

### 抓包看问题

公司内部的一个 golang 中间件报 UDP 连接异常的日志，问题很明显，对端的服务挂了,  自然重启下就可以了. 

哈哈，但让我疑惑的问题是 udp 是如何检测对端挂了? 

```
err:  write udp 172.16.44.62:62651->172.16.0.46:29999: write: connection refused

err:  write udp 172.16.44.62:62651->172.16.0.46:29999: write: connection refused

err:  write udp 172.16.44.62:62651->172.16.0.46:29999: write: connection refused

...
```

udp 协议既没有三次握手，又没有 TCP 那样的状态控制报文，那么如何判定对端的 UDP 端口是否已打开 ?

通过抓包可以发现, 当服务端的端口没有打开时，服务端的系统向客户端返回 `icmp ECONNREFUSED` 报文，表明该连接异常.

通过抓包可以发现返回的协议为 `ICMP`, 但含有源端口和目的端口, 客户端系统解析该报文时，通过五元组找到对应的 socket, 并 errno 返回异常错误，如果客户端陷入等待，则唤醒起来, 设置错误状态.

![](https://xiaorui.cc/image/2020/Jietu20220128-223810.jpg)

(上面是 udp 异常下的 icmp，下面是正常 icmp)

![](https://xiaorui.cc/image/2020/Jietu20220128-223930.jpg)

当 UDP 连接异常时，可以通过 tcpdmp 工具指定 ICMP 协议来抓取该异常报文，毕竟对方是通过 icmp 返回的 ECONNREFUSED.

### 使用 tcpdump 抓包

**请求命令:**

先找到一个可以 ping 通的主机, 然后用 nc 模拟 udp 客户端去请求不存在的端口，出现 `Connection refused`.

```
[root@ocean ~]# nc -vzu 172.16.0.46 8888
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Connected to 172.16.0.46:8888.
Ncat: Connection refused.
```

**抓包信息如下:**

```
[root@ocean ~]# tcpdump -i any icmp -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
17:01:14.075617 IP 172.16.0.46 > 172.16.0.62: ICMP 172.16.0.46 udp port 8888 unreachable, length 37
17:01:17.326145 IP 172.16.0.46 > 172.16.0.62: ICMP 172.16.0.46 udp port 8888 unreachable, length 37
17:01:17.927480 IP 172.16.0.46 > 172.16.0.62: ICMP 172.16.0.46 udp port 8888 unreachable, length 37
17:01:18.489560 IP 172.16.0.46 > 172.16.0.62: ICMP 172.16.0.46 udp port 8888 unreachable, length 37
```

还需要注意的是 telnet 不支持 udp, 只支持 tcp, 建议使用 nc 来探测 udp.

### 各种case的测试

**case小结**

- 当 ip 无法连通时, udp 客户端连接时，通常会显示成功. 
- 当 udp 服务端程序关闭, 但系统还存在时, 对方系统会 `icmp ECONNREFUSE 错误.
- 当对方有操作 iptables udp port drop 时，通常客户端也会显示成功.

**IP 无法联通时:**

```bash
[root@host-46 ~ ]$ ping 172.16.0.65
PING 172.16.0.65 (172.16.0.65) 56(84) bytes of data.
From 172.16.0.46 icmp_seq=1 Destination Host Unreachable
From 172.16.0.46 icmp_seq=2 Destination Host Unreachable
From 172.16.0.46 icmp_seq=3 Destination Host Unreachable
From 172.16.0.46 icmp_seq=4 Destination Host Unreachable
From 172.16.0.46 icmp_seq=5 Destination Host Unreachable
From 172.16.0.46 icmp_seq=6 Destination Host Unreachable
^C
--- 172.16.0.65 ping statistics ---
6 packets transmitted, 0 received, +6 errors, 100% packet loss, time 4999ms
pipe 4

[root@host-46 ~ ]$ nc -vzu 172.16.0.65 8888
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Connected to 172.16.0.65:8888.
Ncat: UDP packet sent successfully
Ncat: 1 bytes sent, 0 bytes received in 2.02 seconds.
```

另外再次明确一点 udp 没有类似 tcp 那样的状态报文, 所以单纯对 UDP 抓包是看不到啥异常信息.

那么当 IP 不通时, 为啥 NC UDP 命令显示成功 ?

### netcat nc udp 的逻辑

为什么当 ip 不连通或者报文被 DROP 时，返回连接成功 ???

因为 nc 默认的探测逻辑很简单，只要在 2 秒钟内没有收到 `icmp ECONNREFUSED` 异常报文, 那么就认为 UDP 连接成功. 😅  

下面是 nc udp 命令执行的过程.

```c
setsockopt(3, SOL_SOCKET, SO_BROADCAST, [1], 4) = 0
connect(3, {sa_family=AF_INET, sin_port=htons(30000), sin_addr=inet_addr("172.16.0.111")}, 16) = 0
select(4, [3], [3], [3], NULL)          = 1 (out [3])
getsockopt(3, SOL_SOCKET, SO_ERROR, [0], [4]) = 0
write(2, "Ncat: ", 6Ncat: )                   = 6
write(2, "Connected to 172.16.0.111:29999."..., 33Connected to 172.16.0.111:29999.
) = 33
sendto(3, "\0", 1, 0, NULL, 0)          = 1

// select 多路复用方法里加入了超时逻辑.
select(4, [3], [], [], {tv_sec=2, tv_usec=0}) = 0 (Timeout)

write(2, "Ncat: ", 6Ncat: )                   = 6
write(2, "UDP packet sent successfully\n", 29UDP packet sent successfully
) = 29
write(2, "Ncat: ", 6Ncat: )                   = 6
write(2, "1 bytes sent, 0 bytes received i"..., 481 bytes sent, 0 bytes received in 2.02 seconds.
) = 48
close(3)                                = 0
```

使用 golang/ python 编写的 UDP 客户端, 给无法连通的地址发 UDP 报文时，其实也不会报错, 这时候通常会认为发送成功. 

还是那句话 UDP 没有 TCP 那样的握手步骤，像 TCP 发送 syn 总得不到回报时, 协议栈会在时间退避下尝试 6 次，当 6 次还得不到回应，内核会给与错误的 errno 值.

### UDP 连接信息

在客户端的主机上, 通过 ss lsof netstat 可以看到 UDP 五元组连接信息.

```bash
[root@host-46 ~ ]$ netstat -tunalp|grep 29999
udp        0      0 172.16.0.46:44136       172.16.0.46:29999       ESTABLISHED 1285966/cccc
```

通常在服务端上看不到 UDP 连接信息, 只可以看到 udp listen 信息 !!!

```
[root@host-62 ~ ]# netstat -tunalp|grep 29999
udp       0      0 :::29999                :::*                                4038720/ss
```

### 客户端重新实例化问题 ?

当 client 跟 server 已连接，server 端手动重启后，客户端无需再次重新实例化连接，可以继续发送数据, 当服务端再次启动后，照样可以收到客户端发来的报文.

udp 本就无握手的过程，他的 udp connect() 也只是在本地创建 socket 信息. 在服务端使用 netstat 是看不到 udp 五元组的 socket.

### Golang 测试代码

**服务端代码:**

```go
package main

import (
    "fmt"
    "net"
)

// UDP 服务端
func main() {
    listen, err := net.ListenUDP("udp", &net.UDPAddr{
        IP:   net.IPv4(0, 0, 0, 0),
        Port: 29999,
    })

    if err != nil {
        fmt.Println("Listen failed, err: ", err)
        return
    }
    defer listen.Close()

    for {
        var data [1024]byte
        n, addr, err := listen.ReadFromUDP(data[:])
        if err != nil {
            fmt.Println("read udp failed, err: ", err)
            continue
        }
        fmt.Printf("data:%v addr:%v count:%v\n", string(data[:n]), addr, n)
    }
}
```

**客户端代码:**

```go
package main

import (
    "fmt"
    "net"
    "time"
)

// UDP 客户端
func main() {
    socket, err := net.DialUDP("udp", nil, &net.UDPAddr{
        IP:   net.IPv4(172, 16, 0, 46),
        Port: 29999,
    })
    if err != nil {
        fmt.Println("连接UDP服务器失败，err: ", err)
        return
    }
    defer socket.Close()

    for {
        time.Sleep(1e9 * 2)
        sendData := []byte("Hello Server")
        _, err = socket.Write(sendData)
        if err != nil {
            fmt.Println("发送数据失败，err: ", err)
            continue
        }

        fmt.Println("已发送")
    }
}
```

###  总结

当 udp 服务端的机器可以连通且无异常时，客户端通常会显示成功。但当有异常时，会有以下的情况:

- 当 ip 地址无法连通时, udp 客户端连接时，通常会显示成功. 

- 当 udp 服务端程序关闭, 但系统还存在时, 对方系统通过 `icmp ECONNREFUSE` 返回错误，客户端会报错.

- 当对方有操作 iptables udp port drop 时，客户端也会显示成功.

- 客户端和服务端互通数据，当服务进程挂了时，UDP 客户端不能立马感知关闭状态，只有当再次发数据时才会被对方系统回应 `icmp ECONNREFUSE` 异常报文, 客户端才能感知对方挂了.