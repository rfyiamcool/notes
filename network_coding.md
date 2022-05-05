### 前言

😅 年前的时候, 面向全公司做了一场关于网络编程的分享, 其实公司内部很多人对这个话题很感兴趣，刚好我近几年也一直在搞网络编程相关的开发，积攒了不少网络编程的开发经验. 就这么做了 70 多页的PPT，用了上下两场次来讲完. 其实这个话题还是很枯燥的，前期大家还能跟上，后面随着内容的深入和各种原理的讲述，大家其实早已躁动不安了.

我把 ppt 扔到了 github 中, 如果大家在观看 ppt 中有不理解的地方，可以到 [https://github.com/rfyiamcool/share_ppt](https://github.com/rfyiamcool/share_ppt) 这里提问题 issue .

### 内容

**按照演讲目录来说，分了几块内容.**

- socket原理
  - 收包原理
  - 发包原理
  - 阻塞非阻塞
  - 同步异步
  - 五种io模型
- io多路复用
  - select
  - poll
  - epoll
    - 数据结构组成
    - 如何使用 epoll 的那几个方法
    - 从底层来讲解 epoll 的实现原理
    - epolloneshot 的场景
    - 水平触发和边缘触发到底是怎么一回事, 各种case来描述
    - 社区中常见的服务端使用 epoll 哪种触发模型
    - epoll 的开发技巧
- aio 到底是怎么一回事？ 存在的问题
- 当前社区比较流行的 网络编程 模型
  - 新线程模型
  - 单多路复用 + 业务线程池模型
  - prefork 模型
  - reactor
  - proactor
  - golang 
    - golang netpoll
    - golang raw epoll
    - golang reactor
- 常见的网络编程问题
  - 常见的读写返回值的处理方式
  - epoll 的惊群问题
  - 客户端如何异步连接
  - 粘包半包的问题
  - socket close vs shutdown
  - tcp rst 问题
  - 半关闭
  - 如何设置心跳，为何需要应用层心跳
  - time-wait的问题
  - tcp 连接队列是怎么一回事，如何配置
  - 移动弱网络的表现及优化
  - 传统 tcp 的问题, 拥塞控制
  - 可靠udp通信 kcp 的优缺点
  - 连接端口的问题
  - server端挂了，系统挂了，路由器挂了分别会引发什么问题
  - 如何优雅升级
  - 如何处理各种的网络异常问题

### 地址

完整 ppt 的下载地址 :

[https://github.com/rfyiamcool/share_ppt#%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B%E9%82%A3%E4%BA%9B%E4%BA%8B%E5%84%BF](https://github.com/rfyiamcool/share_ppt#%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B%E9%82%A3%E4%BA%9B%E4%BA%8B%E5%84%BF)

> 记得点 github star 哈 ... 

### 截图预览

ppt 的页数有 70 页，这里只是传了部分的预览图，想看完整的 ppt 可以到github中看。

![](https://xiaorui.cc/image/network_coding/Jietu20220505-222440.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222459.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222516.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222544.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222551.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222603.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222612.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222622.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222647.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222658.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222707.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222717.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222732.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222739.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222804.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222817.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222825.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222842.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222904.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222911.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222920.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222930.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-222952.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223006.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223016.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-231748.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223033.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223053.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223106.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223121.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223141.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-231500.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223159.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223215.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223232.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-231628.jpg)
![](https://xiaorui.cc/image/network_coding/Jietu20220505-223309.jpg)