# golang gomaxprocs 不匹配引起 runtime 调度性能损耗

先前在社区里分享了关于golang行情推送的分享，有人针对ppt的内容问了我两个问题，一个是在docker下golang的gomaxprocs初始化混乱问题，另一个是golang runtime.gomaxprocs配置多少为合适？

![](http://xiaorui.cc/wp-content/uploads/2020/01/aa.jpg)

分享下上面行情推送的ppt地址，有兴趣的可以看看。 http://xiaorui.cc/?p=6250

## golang runtime

golang的runtime调度是依赖pmg的角色抽象，p为逻辑处理器，m为执行体(线程)，g为协程。p的runq队列中放着可执行的goroutine结构。golang默认p的数量为cpu core数目，比如物理核心为8cpu core，那么go processor的数量就为8。另外，同一时间一个p只能绑定一个m线程，pm绑定后自然就找g和运行g。

那么增加processor的数量，是否可以用来加大runtime对于协程的调度吞吐？

大多golang的项目偏重网络io，network io在netpoll设计下都是非阻塞的，所涉及到的syscall不会阻塞。如果是cpu密集的业务，增加多个processor也没用，毕竟cpu计算资源就这些，居然还想着来回的切换？ 😅 所以，多数场景下单纯增加processor是没什么用的。

当然，话不绝对，如果你的逻辑含有不少的cgo及阻塞syscall的操作，那么增加processor还是有效果的，最少在我实际项目中有效果。原因是这类操作有可能因为过慢引起阻塞，在阻塞期间的p被该mg绑定一起，其他m无法获取p的所有权。虽然在findrunnable steal机制里，其他p的m可以偷该p的任务，但在解绑p之前终究还是少了一条并行通道。另外，runtime的sysmon周期性的检查长时间阻塞的pmg， 并抢占并解绑p。

## golang在docker下问题

在微服务体系下服务的部署通常是放在docker里的。一个宿主机里跑着大量的不同服务的容器，为了避免资源冲突，通常会合理的对每个容器做cpu资源控制。比如给一个golang服务的容器限定了2cpu core的资源，容器内的服务不管怎么折腾，也确实只能用到大约2个cpu core的资源。

但golang初始化processor数量是依赖/proc/cpuinfo信息的，容器内的cpuinfo是跟宿主机一致的，这样导致容器只能用到2个cpu core，但golang初始化了跟物理cpu core相同数量的processor。

```
// xiaorui.cc

限制2核左右
root@xiaorui.cc:~# docker run -tid --cpu-period 100000 --cpu-quota 200000 ubuntu

容器内
root@a4f33fdd0240:/# cat /proc/cpuinfo| grep "processor"| wc -l
48
```

## runtime processor多了会出现什么问题？

一个runtime findrunnable时产生的损耗，另一个是线程引起的上下文切换。

runtime的findrunnable方法是解决m找可用的协程的函数，当从绑定p本地runq上找不到可执行的goroutine后，尝试从全局链表中拿，再拿不到从netpoll和事件池里拿，最后会从别的p里偷任务。全局runq是有锁操作，其他偷任务使用了atomic原子操作来规避futex竞争下陷入切换等待问题，但lock free在竞争下也会有忙轮询的状态，比如不断的尝试。

```go
// xiaorui.cc

    // 全局 runq
    if sched.runqsize != 0 {
        lock(&sched.lock)
        gp := globrunqget(_p_, 0)
        unlock(&sched.lock)
        if gp != nil {
            return gp, false
        }
    }
...
    // 尝试4次从别的p偷任务
     for i := 0; i < 4; i++ {
        for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
            if sched.gcwaiting != 0 {
                goto top
            }
            stealRunNextG := i > 2 // first look for ready queues with more than 1 g
            if gp := runqsteal(_p_, allp[enum.position()], stealRunNextG); gp != nil {
                return gp, false
            }
        }
    }
...
```

通过godebug可以看到全局队列及各个p的runq里等待调度的任务量。有不少p是空的，那么势必会引起steal偷任务。另外，runqueue的大小远超其他p的总和，说明大部分任务在全局里，全局又是把大锁。

![](http://xiaorui.cc/wp-content/uploads/2020/01/jjj.jpg)

随着调多runtime processor数量，相关的m线程自然也就跟着多了起来。linux内核为了保证可执行的线程在调度上雨露均沾，按照内核调度算法来切换就绪状态的线程，切换又引起上下文切换。上下文切换也是性能的一大杀手。findrunnable的某些锁竞争也会触发上下文切换。

下面是我这边一个行情推送服务压测下的vmstat监控数据。首先把容器的的cpu core限制为8，再先后测试processor为8和48的情况。图的上面是processor为8的情况，下面为processor为48的情况。看图可以直观的发现当processor调大后，上下文切换明显多起来，另外等待调度的线程也多了。

![](http://xiaorui.cc/wp-content/uploads/2020/01/1111.jpg)

另外从qps的指标上也可以反映多processor带来的性能损耗。通过下图可以看到当runtime.GOMAXPROCS为固定的cpu core数时，性能最理想。后面随着processor数量的增长，qps指标有所下降。

![](http://xiaorui.cc/wp-content/uploads/2020/01/qps-2.jpg)

通过golang tool trace可以分析出协程调度等待时间越来越长了。

![](http://xiaorui.cc/wp-content/uploads/2020/01/aaa.jpg)

## 解决docker下的golang gomaxprocs校对问题

有两个方法可以准确校对golang在docker的cpu获取问题。

要么在k8s pod里加入cpu限制的环境变量。容器内的golang服务在启动时获取关于cpu的信息。

要么解析cpu.cfs_period_us和cpu.cfs_quota_us配置来计算cpu资源。社区里有不少这类的库可以使用，uber的automaxprocs可以兼容docker的各种cpu配置。 

[https://github.com/uber-go/automaxprocs](https://github.com/uber-go/automaxprocs)

## 总结

建议gomaxprocs配置为cpu core数量就可以了，go默认就是这个配置，无需再介入。如果涉及到阻塞syscall，可以适当的调整gomaxprocs大小，但一定要用指标数据说话 !
