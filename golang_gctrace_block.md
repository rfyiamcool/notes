# golang gctrace 引起 runtime 调度阻塞

这是一个奇葩的问题。当你开启GODEBUG=gctrace=1，并且日志是重定向到文件，那么有概率会造成runtime调度阻塞。😅

## 查找问题

开始业务方说我们的grpc sidecar时不时发生高时延抖动，我对比了监控上的性能数据，貌似是磁盘io引起的，我迅速调高了日志级别及修改日志库为异步写入模式。接口的时延抖动问题确实减少了，但依旧还是出现。

通过项目代码变更得知，一同事在部署脚本里加入了gctrace监控。开始不觉得是这个问题引起的，但近一段时间也就这一个commit提交，我就尝试回滚代码，问题居然就这么解决了。

## 分析gc gctrace源码

我们先来分析下golang gc gctrace相关的代码，我们知道go 1.12的stop the world会发生mark的两个阶段，一个是mark setup开始阶段，另一个就是mark termination收尾阶段。gc在进行到gcMarkTermination时，会stop the world，然后判断是否需要输出gctrace的调试日志。

简单说，这个打印输出的过程是在stop the world里。默认是打印到pts伪终端上，这个过程是纯内存操作，理论是很快的。

但如果在开启gctrace时进行文件重定向，那么他的操作就是文件io操作了。如果这时候你的服务里有大量的磁盘io的操作，本来写page buffer的操作，会触发阻塞flush磁盘。那么这时候go gctrace打印日志是在开启stop the world之后操作的，因为磁盘繁忙，只能是等待磁盘io操作完，这里的stw会影响runtime对其他协程的调度。

```go
// xiaorui.cc
func gcMarkTermination(nextTriggerRatio float64) {
    // World is stopped  已经开启了stop the world
    ...
    // Print gctrace before dropping worldsema.

    if debug.gctrace &gt; 0 {
		util := int(memstats.gc_cpu_fraction * 100)

		var sbuf [24]byte
		printlock()
		print("gc ", memstats.numgc,
			" @", string(itoaDiv(sbuf[:], uint64(work.tSweepTerm-runtimeInitTime)/1e6, 3)), "s ",
			util, "%: ")
		prev := work.tSweepTerm
		for i, ns := range []int64{work.tMark, work.tMarkTerm, work.tEnd} {
			if i != 0 {
				print("+")
			}
			print(string(fmtNSAsMS(sbuf[:], uint64(ns-prev))))
			prev = ns
		}
		print(" ms clock, ")
		for i, ns := range []int64{sweepTermCpu, gcController.assistTime, gcController.dedicatedMarkTime + gcController.fractionalMarkTime, gcController.idleMarkTime, markTermCpu} {
			if i == 2 || i == 3 {
				// Separate mark time components with /.
				print("/")
			} else if i != 0 {
				print("+")
			}
			print(string(fmtNSAsMS(sbuf[:], uint64(ns))))
		}
		print(" ms cpu, ",
			work.heap0&gt;&gt;20, "-&gt;", work.heap1&gt;&gt;20, "-&gt;", work.heap2&gt;&gt;20, " MB, ",
			work.heapGoal&gt;&gt;20, " MB goal, ",
			work.maxprocs, " P")
		if work.userForced {
			print(" (forced)")
		}
		print("\n")
		printunlock()
	}

	semrelease(&amp;worldsema)  // 关闭stop the world

        ...
}
```

## 解决方法

线上就不应该长期的去监控gctrace的日志。另外需要把较为频繁的业务日志进行采样输出。前面有说，我们第一个解决方法就修改日志的写入模式，当几千个协程写日志时，不利于磁盘的高效使用和性能，可以借助disruptor做日志缓存，然后由独立的协程来写入日志。

除此之外，还发现一个问题，开始时整个golang进程开了几百个线程，这是由于过多的写操作超过磁盘瓶颈，继而触发了disk io阻塞。这时候golang runtime sysmon检测到syscall超时，继而解绑mp，接着实例化新的m线程进行绑定p。记得以前专门写过文章介绍过golang线程爆满的问题。

## 总结

我们可以想到如果函数gcMarkTermination输出日志放在关闭stop the world后面，这样就不会影响runtime调度。当然，这个在线上重定向go gctrace的问题本来就很奇葩。😅

一个腾讯的朋友说跟我说，go夜读里有个腾讯小哥做了runtime分享，中间也遇到了同样的问题，针对该问题也做了分析。看来这奇葩事情不止是我遇到了。😁