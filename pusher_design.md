### 前言

面向全公司做了一场技术分享，主题是 `<< 分布式消息推送系统 >> `. 该系统的是我近几个月的工作内容，开发的过程中有一些心得和坑分享给大家. 重点描述了分布式消息推送的架构设计.

- 如何保证消息的可靠性 ?
- 如何实现消息的实时性 ?
- 如何维护连接状态 ?
- 如何维护客户端路由器 ?
- 如何适配异地多活系统 ?
- 如何实现消息重试 ?
- 如何实现离线消息 ?
- 如何使用raw epoll替换netpoll实现接入服务 ?
- 如何分布式压测 ?
- 如何做到全方位指标的可观测性 ?
- 通过二级缓存减少时延
- golang在推送系统中的优化
- golang在推送系统中的编码技巧

公司在市场上的餐饮应用和设备本就多达将近几十种, 近期公司对众多的商家门店又推广了一些辅助的设备和APP，所以一个集团下的一个门店最少有一个设备或APP, 设备可以理解为一个店铺里的一个端. 业务属性是餐饮行业, 消息的种类繁多，消息的频次也多, 并发的特征很明显，在饭点时间出现大量的消息推送和连接.

到 `2021-05` 为止，公司的在线客户端稳定在 70w 个连接, 突发峰值可能破 85w 个左右, 所有节点加起来每天推送消息的总量在 6 个亿左右.

这么大的量线上肯定采用集群式部署, 足足有10台高性能主机, 虽然跟其他服务是混部模式，但每个节点无需承载太多的流量, 推送服务本身也有过载保护机制，每个主机最多被调度到20w条连接。听着这部署规模是稳妥的. 但作为一个优秀的程序员，我们依旧想让推送系统有更好的性能，保证单个服务最少可以抗30w的连接. 

> 有时间再补充每页ppt内容，很多时候知道我没时间补充文稿，所以尽量用图片代替文字来描述枯燥的内容。

如果 ppt 中有你不理解的地方，大家可以到 [https://github.com/rfyiamcool/share_ppt](https://github.com/rfyiamcool/share_ppt) 这里提问题 issue .

### 内容

**按照演讲目录来说，分了几块内容.**

- 什么是推送系统
- 推送系统的架构
- 性能调优
- 压力测试
- 运维部署

### 地址

完整 ppt 的下载地址 :

[https://github.com/rfyiamcool/share_ppt/blob/master/message_pusher.pdf](https://github.com/rfyiamcool/share_ppt/blob/master/message_pusher.pdf)

> 记得点 github star 哈 ... 

### 截图预览

ppt 的页数有 50 页，只是传了部分的预览图，想看完整的 ppt 可以到github中看。

![](https://xiaorui.cc/image/2020/message_pusher%204%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%205%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%206%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%209%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2010%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2011%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2012%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2013%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2014%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2015%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2016%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2017%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2018%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2019%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2020%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2022%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2023%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2024%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2025%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2026%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2027%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2028%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2029%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2030%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2031%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2032%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2033%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2034%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2035%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2036%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2037%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2038%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2039%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2041%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2042%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2043%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2044%204:8:2021.jpeg)
![](https://xiaorui.cc/image/2020/message_pusher%2045%204:8:2021.jpeg)