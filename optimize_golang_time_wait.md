## 高并发服务遇redis瓶颈引发time-wait事故

### 摘要

元旦期间 `订单业务线` 告知 `推送系统` 无法正常收发消息，作为推送系统维护者的我正外面潇洒，无法第一时间回去，直接让ops帮忙重启服务，一切好了起来，重启果然是个大杀器。由于推送系统本身是分布式部署，消息有做各种的可靠性策略，所以重启是不会丢失消息事件的。

😅 事后通过日志分析有大量的redis的报错，十分钟内有16w次的错误。日志的错误是 `connect: cannot assign requested address` 。该错误不是推送服务内部及redis库返回的error，而是系统回馈的errno错误。

这个错误是由于无法申请可用地址引起的，就是无法申请到可用的socket。

话说，元旦当天在线数和订单量确实大了不少，往常推送系统的长连接客户端在35w，这次峰值飙到`50w`左右, 集群共6个节点，其中有4个节点每个都抗了9w+的长连接。另外，推送的消息量也随之翻倍。

![https://gitee.com/rfyiamcool/image/raw/master/2020/20210103201357.png](https://gitee.com/rfyiamcool/image/raw/master/2020/20210103201357.png)

### 分析

下面是kibana日志的统计，出错的时间区间里有近16w次的redis报错。

![https://gitee.com/rfyiamcool/image/raw/master/2020/20210103193642.png](https://gitee.com/rfyiamcool/image/raw/master/2020/20210103193642.png)

下面是出问题节点的TCP连接状况，可以看到established在6w，而`time-wait` 连接干到2w多个。

![https://gitee.com/rfyiamcool/image/raw/master/2020/20210103193737.png](https://gitee.com/rfyiamcool/image/raw/master/2020/20210103193737.png)

为什么会产生这么多time-wait ？ 谁主动关闭就就有time-wait，但推送系统除了协议解析失败之外，其余情况都不会主动close客户端的，哪怕是鉴权失败和弱网络客户端写缓冲爆满，事后通过日志也确定了不是由推送系统自身产生的tw。

另外，linux主机被ops交付时应该有做内核调优初始化的，在开启tw_reuse参数后，time-wait是可以复用的。难道是没开启reuse？

查看sysctl.conf的内核参数得知，果然`tcp_tw_reuse`参数没有打开，不能快速的复用还处在time-wait状态的地址，只能等待 time-wait的超时关闭，rfc协议里规定等待2分钟左右，开启tw_reuse可在1s后复用该地址。另外`ip_local_port_range`端口范围也不大，缩短了可用的连接范围。

```bash
sysctl  -a|egrep "tw_reuse|timestamp|local_port"

net.ipv4.ip_local_port_range = 35768	60999
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_tw_reuse = 0
```

所以，由于没有可用地址才爆出 `connect: cannot assign requested address` 错误。

### 内在问题

**追究问题**

上面是表象问题，来查查为什么会有这么多的time-wait ？再说一遍，通常哪一端主动close fd，哪一端就会产生time-wait。事后通过 netstat 得知time-wait连接基本是来自redis主机。

下面是推送代码中的连接池配置，空闲连接池只有50，最大可以new的连接可以到500个。这代表当有大量请求时，企图先从size为50的连接池里获取连接，如果拿不到连接则new一个新连接，连接用完了后需要归还连接池，如果这时候连接池已经满了，那么该连接会主动进行close关闭。

```bash
MaxIdle   = 10
MaxActive = 500
Wait      = false
```

除此之外，还发现一个问题。有几处redis的处理逻辑是异步的，比如每次收到心跳包都会go一个协程去更新redis, 这也加剧了连接池的抢夺，改为同步代码。这样在一个连接上下文中同时只对一个redis连接操作。

**解决方法**

调大golang redis client的maxIdle连接池大小，避免了大并发下无空闲连接而新建连接和池子爆满又不能归还连接的尴尬场面。当pool wait 为true时，意味着如果空闲池中没有可用的连接且当前Active连接数 > MaxIdle，则阻塞等待可用的连接。反之直接返回 "connection pool exhausted" 错误。

```bash
MaxIdle   = 300
MaxActive = 400
Wait      = true
```

###  redis的qps性能瓶颈

redis的性能一直是大家所称赞的，在不使用redis 6.0 multi io thread下，QPS一般可以在13w左右，如果使用多指令和pipeline的话，可以干到40w的OPS命令数，当然qps还是在12w-13w左右。

> Redis QPS高低跟redis版本和cpu hz、cache存在正比关系

根据我的经验，在内网环境下且已实例化连接对象，单条redis指令请求耗时通常在0.2ms左右，200us微妙已经够快了，但为什么还会因redis client连接池无空闲连接而建立新连接的情况？

通过grafana监控分析redis集群，发现有几个节点QPS已经到了Redis单实例性能瓶颈，QPS干到了近15w左右。难怪不能快速处理来自业务的redis请求。这瓶颈必然会影响请求的时延。请求的时延都高了，连接池不能及时返回连接池，所以就造成中描述的问题。总之，业务流量的暴增引起的一系列问题。

![https://gitee.com/rfyiamcool/image/raw/master/2020/20210103222040.png](https://gitee.com/rfyiamcool/image/raw/master/2020/20210103222040.png)

发现问题，那么就要解决问题，redis的qps优化方案有两步：

- 扩容redis节点，迁移slot使其分担流量
- 尽量把程序中redis的请求改成批量模式

增加节点容易，批量也容易。期初在优化推送系统时，已经把同一个逻辑中的redis操作改为批量模式了。但问题来了，很多的redis操作在不同的逻辑块里面，没法合成一个pipeline。

然后做了进一步的优化，把不同逻辑中的redis请求合并到一个pipeline里，优点在于提高了redis的吞吐，减少了socket系统调用，网络中断开销，缺点是增加了逻辑复杂度，使用channal管道做队列及通知增加了runtime调度开销，pipeline worker触发条件是满足3个command和5ms超时，定时器采用分段的时间轮。通过压力测试cpu减少了 `4%` 左右，但redis qps降到7w左右。

![](https://gitee.com/rfyiamcool/image/raw/master/2020/Jietu20210104-151628.jpg)

这个方案来自我在上家公司做推送系统的经验，有兴趣的朋友可以看看PPT，内涵不少高并发经验。[分布式推送系统设计与实现](https://github.com/rfyiamcool/share_ppt#%E5%88%86%E5%B8%83%E5%BC%8F%E8%A1%8C%E6%83%85%E6%8E%A8%E9%80%81%E7%B3%BB%E7%BB%9Fgolang)

### 总结

推送系统设计之初是预计15w的长连接数，稳定后再无优化调整，也一直稳定跑在线上。后面随着业务的暴涨，长连接数也一直跟着暴涨，现在日常稳定在35w，出问题时暴到50w，我们没有因为业务暴增进行整条链路压测及优化。

话说，如果对推送系统平时多上点心也不至于出这个问题。我曾经开发过相对高规格的推送系统，而现在公司的推送系统我是后接手的，由于它的架子一般，但业务性又太强太强，所以就没有推到重构，仅仅在此架子上添添补补，所以对该系统一直不上心。