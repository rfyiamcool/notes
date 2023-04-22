# golang runtime preempt scheduler 基于信号的抢占式调度的实现原理

![https://xiaorui.cc/wp-content/uploads/2020/02/Jietu20200213-232930_Fotor.jpg](https://xiaorui.cc/wp-content/uploads/2020/02/Jietu20200213-232930_Fotor.jpg)

golang在之前的版本中已经实现了抢占调度，不管是陷入到大量计算还是系统调用，大多可被sysmon扫描到并进行抢占。但有些场景是无法抢占成功的。比如轮询计算 `for { i++ }` 等，这类操作无法进行newstack、morestack、syscall，所以无法检测 `stackguard0 = stackpreempt`。

go team已经意识到抢占是个问题，所以在1.14中加入了基于信号的协程调度抢占。原理是这样的，首先注册绑定 SIGURG 信号及处理方法runtime.doSigPreempt，sysmon会间隔性检测超时的p，然后发送信号，m收到信号后休眠执行的goroutine并且进行重新调度。

## 对比测试

```go
// xiaorui.cc

package main

import (
    "runtime"
)

func main() {
    runtime.GOMAXPROCS(1)

    go func() {
        panic("already call")
    }()

    for {
    }
}
```

上面的测试思路是先针对GOMAXPROCS的p配置为1，这样就可以规避并发而影响抢占的测试，然后go关键字会把当前传递的函数封装协程结构,扔到runq队列里等待runtime调度，由于是异步执行，所以就执行到for死循环无法退出。

go1.14是可以执行到panic，而1.13版本一直挂在死循环上。那么在go1.13是如何解决这个问题？ 要么并发加大，要么执行一个syscall，要么执行复杂的函数产生morestack扩栈。对比go1.13版，通过strace可以看到go1.14多了一步发送信号中断。这看似就是文章开头讲到的基于信号的抢占式调度了。

![https://xiaorui.cc/wp-content/uploads/2020/02/Jietu20200213-212221_Fotor.jpg](https://xiaorui.cc/wp-content/uploads/2020/02/Jietu20200213-212221_Fotor.jpg)

## 源码分析

以前写过文章来分析go sysmon() 的工作，在新版go 1.14里其他功能跟以前一样，只是加入了信号抢占。

### 怎么注册的sigurg信号？

```go
// xiaorui.cc

const sigPreempt = _SIGURG

func initsig(preinit bool) {
    for i := uint32(0); i < _NSIG; i++ {
        fwdSig[i] = getsig(i)
        ,,,
        setsig(i, funcPC(sighandler)) // 注册信号对应的回调方法
    }
}

func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
    ,,,
    if sig == sigPreempt {  // 如果是抢占信号
        // Might be a preemption signal.
        doSigPreempt(gp, c)
    }
    ,,,
}

// 执行抢占
func doSigPreempt(gp *g, ctxt *sigctxt) {
    if wantAsyncPreempt(gp) && isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()) {
        // Inject a call to asyncPreempt.
        ctxt.pushCall(funcPC(asyncPreempt))  // 执行抢占的关键方法
    }

    // Acknowledge the preemption.
    atomic.Xadd(&gp.m.preemptGen, 1)
}
```

go在启动时把所有的信号都注册了一遍，包括可靠的信号。(截图为部分)

![https://xiaorui.cc/wp-content/uploads/2020/02/Jietu20200214-014900_Fotor.jpg](https://xiaorui.cc/wp-content/uploads/2020/02/Jietu20200214-014900_Fotor.jpg)

### 由谁去发起检测抢占?

go1.14之前的版本是是由sysmon检测抢占，到了go1.14当然也是由sysmon操作。runtime在启动时会创建一个线程来执行sysmon，为什么要独立执行？ sysmon是golang的runtime系统检测器，sysmon可进行forcegc、netpoll、retake等操作。拿抢占功能来说，如sysmon放到pmg调度模型里，每个p上面的goroutine恰好阻塞了，那么还怎么执行抢占？

所以sysmon才要独立绑定运行，就上面的脚本在测试运行的过程中，虽然看似阻塞状态，但进行strace可看到sysmon在不断休眠唤醒操作。sysmon启动后会间隔性的进行监控，最长间隔10ms，最短间隔20us。如果某协程独占P超过10ms，那么就会被抢占！

sysmon依赖schedwhen和schedtick来记录上次的监控信息，schedwhen记录上次的检测时间，schedtick来区分调度时效。比如sysmon在两次监控检测期间，已经发生了多次runtime.schedule协程调度切换，每次调度时都会更新schedtick值。所以retake发现sysmontick.schedtick值不同时重新记录schedtick。

源码位置: `runtime/proc.go`

```go
// xiaorui.cc

func main() {
    g := getg()
    ,,,
    if GOARCH != "wasm" {
        systemstack(func() {
            newm(sysmon, nil)
        })
    }
    ,,,
}

func schedule() {
    ,,,
    execute(gp, inheritTime)
}

func execute(gp *g, inheritTime bool) {
    if !inheritTime {
        _g_.m.p.ptr().schedtick++
    }
    ,,,
}

func sysmon(){
    ,,,
         // retake P's blocked in syscalls
         // and preempt long running G's
         if retake(now) != 0 {
             idle = 0
         } else {
             idle++
         }
    ,,,
}

// 记录每次检查的信息
type sysmontick struct {
    schedtick   uint32
    schedwhen   int64
    syscalltick uint32
    syscallwhen int64
}

const forcePreemptNS = 10 * 1000 * 1000 // 抢占的时间阈值 10ms

func retake(now int64) uint32 {
    n := 0
    lock(&allpLock)
    for i := 0; i < len(allp); i++ {
        _p_ := allp[i]
        if _p_ == nil {
            continue
        }
        pd := &_p_.sysmontick
        s := _p_.status
        if s == _Prunning || s == _Psyscall {
            // Preempt G if it's running for too long.
            t := int64(_p_.schedtick)
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now // 记录当前检测时间
            // 上次时间加10ms小于当前时间，那么说明超过，需进行抢占。
            } else if pd.schedwhen+forcePreemptNS <= now {
                preemptone(_p_)
            }
        }

        // 下面省略掉慢系统调用的抢占描述。
        if s == _Psyscall {
            // 原子更为p状态为空闲状态
            if atomic.Cas(&_p_.status, s, _Pidle) {
                ,,,
                handoffp(_p_)  // 强制卸载P, 然后startm来关联
            }
        ,,,
} 

func preemptone(_p_ *p) bool {
    mp := _p_.m.ptr()
    ,,,

    gp.preempt = true

    ,,,
    gp.stackguard0 = stackPreempt

    // Request an async preemption of this P.
    if preemptMSupported && debug.asyncpreemptoff == 0 {
        _p_.preempt = true
        preemptM(mp)
    }

    return true
}
```

### 发送SIGURG信号？

源码位置: `signal_unix.go`

```go
// 给m发送sigurg信号
func preemptM(mp *m) {
    if !pushCallSupported {
        // This architecture doesn't support ctxt.pushCall
        // yet, so doSigPreempt won't work.
        return
    }
    if GOOS == "darwin" && (GOARCH == "arm" || GOARCH == "arm64") && !iscgo {
        return
    }
    signalM(mp, sigPreempt)
}
```

### 收到sigurg信号后如何处理 ?

preemptPark方法会解绑mg的关系，封存当前协程，继而重新调度runtime.schedule()获取可执行的协程，至于被抢占的协程后面会去重启。

goschedImpl操作就简单的多，把当前协程的状态从_Grunning正在执行改成 _Grunnable可执行，使用globrunqput方法把抢占的协程放到全局队列里，根据pmg的协程调度设计，globalrunq要后于本地runq被调度。

源码位置: `runtime/preempt.go`

```go
// xiaorui.cc

//go:generate go run mkpreempt.go

// asyncPreempt saves all user registers and calls asyncPreempt2.
//
// When stack scanning encounters an asyncPreempt frame, it scans that
// frame and its parent frame conservatively.
func asyncPreempt()

//go:nosplit
func asyncPreempt2() {
    gp := getg()
    gp.asyncSafePoint = true
    if gp.preemptStop {
        mcall(preemptPark)
    } else {
        mcall(gopreempt_m)
    }
    gp.asyncSafePoint = false
}
```

源码位置: `runtime/proc.go`

```go
// preemptPark parks gp and puts it in _Gpreempted.
//
//go:systemstack
func preemptPark(gp *g) {
    ,,,
    status := readgstatus(gp)
    if status&^_Gscan != _Grunning {
        dumpgstatus(gp)
        throw("bad g status")
    }
    ,,,
    schedule()
}

func goschedImpl(gp *g) {
    status := readgstatus(gp)
    ,,,
    casgstatus(gp, _Grunning, _Grunnable)
    dropg()
    lock(&sched.lock)
    globrunqput(gp)
    unlock(&sched.lock)

    schedule()
}
```

源码解析粗略的分析完了，还有一些细节不好读懂，但信号抢占实现的大方向摸的89不离10了。

### 抢占是否影响性能 ？

抢占分为 Prunning 和 Psyscall.

- Psyscall 抢占通常是由于阻塞性系统调用引起的，比如磁盘io、cgo 等。
- Prunning 抢占通常是由于一些类似死循环的计算逻辑引起的，比如 for { i ++ } 。

过度的发送信号来中断m进行抢占多少会影响性能的，主要是软中断和上下文切换。在平常的业务逻辑下，多数不会这类阻塞问题。

### 慢系统调度的错误处理？

EINTR 错误通常是由于被信号中断引起的错误，比如在执行 epollwait、accept、read、write 等操作时，收到信号，那么该系统调用会被打断中断，然后去执行信号注册的回调方法，完事后会返回eintr错误。

下面是 golang 源码中的处理方法，由于 golang 的 netpoll 设计使多数的网络 io 相关的 syscall 操作非阻塞化，所以就只有 epollwait 有该问题。
下面是 golang epollwait 的处理方法，epoll 在 fd 设为非阻塞后，基本不会阻塞，但 epollwait 依旧是阻塞方法，当然你可以 timeout 设为 0，但不能这么用。

```go
// xiaorui.cc

func netpoll(delay int64) gList {
    ,,,
    var events [128]epollevent
retry:
    n := epollwait(epfd, &events[0], int32(len(events)), waitms)
    if n < 0 {
        if n != -_EINTR {
            println("runtime: epollwait on fd", epfd, "failed with", -n)
            throw("runtime: netpoll failed")
        }
        goto retry
    }
    ,,,
}

func netpollBreak() {
    for {
        var b byte
        n := write(netpollBreakWr, unsafe.Pointer(&b), 1)
        if n == 1 {
            break
        }
        if n == -_EINTR {
            continue
        }
        ,,,
    }
}
```

通常需要手动来解决 EINTR 的错误问题，虽然可通过 SA_RESTART 来重启被中断的系统调用，但不管是 syscall 兼容和业务上有可能出现偏差。

```go
// xiaorui.cc

// epoll_wait
if(  -1 == epoll_wait() )
{
    if(errno!=EINTR)
    {
          return -1;
    }
}

// read 
again:
          if ((n = read(fd， buf， BUFFSIZE)) < 0) {
             if (errno == EINTR)
                  goto again;
          }
```

配置 SA_RESTART 后，线程被中断后还可继续执行被中断的系统调用。

```go
// xiaorui.cc

--- SIGINT {si_signo=SIGINT, si_code=SI_KERNEL, si_value={int=0, ptr=0x100000000}} ---
rt_sigreturn()                          = -1 EINTR (Interrupted system call)
futex(0x1b97a30, FUTEX_WAIT_PRIVATE, 0, NULL) = ? ERESTARTSYS (To be restarted if SA_RESTART is set)
--- SIGINT {si_signo=SIGINT, si_code=SI_KERNEL, si_value={int=0, ptr=0x100000000}} ---
rt_sigreturn()                          = -1 EINTR (Interrupted system call)
...
```

## 信号的原理？

我们对一个进程发送信号后，内核把信号挂载到目标进程的信号 pending 队列上去，然后进行触发软中断设置目标进程为running状态。当进程被唤醒或者调度后获取CPU后，才会从内核态转到用户态时检测是否有signal等待处理，等进程处理完后会把相应的信号从链表中去掉。

通过kill -l拿到当前系统支持的信号列表，1-31为不可靠信号，也是非实时信号，信号有可能会丢失，比如发送多次相同的信号，进程只能收到一次。

```go
// xiaorui.cc

// Print a list of signal names.  These are found in /usr/include/linux/signal.h

kill -l

1) SIGHUP     2) SIGINT     3) SIGQUIT     4) SIGILL     5) SIGTRAP
6) SIGABRT     7) SIGBUS     8) SIGFPE     9) SIGKILL    10) SIGUSR1
11) SIGSEGV    12) SIGUSR2    13) SIGPIPE    14) SIGALRM    15) SIGTERM
16) SIGSTKFLT    17) SIGCHLD    18) SIGCONT    19) SIGSTOP    20) SIGTSTP
21) SIGTTIN    22) SIGTTOU    23) SIGURG    24) SIGXCPU    25) SIGXFSZ
26) SIGVTALRM    27) SIGPROF    28) SIGWINCH    29) SIGIO    30) SIGPWR
31) SIGSYS  
```

在Linux中的posix线程模型中，线程拥有独立的进程号，可以通过getpid()得到线程的进程号，而线程号保存在pthread_t的值中。而主线程的进程号就是整个进程的进程号，因此向主进程发送信号只会将信号发送到主线程中去。如果主线程设置了信号屏蔽，则信号会投递到一个可以处理的线程中去。

注册的信号处理函数都是线程共享的，一个信号只对应一个处理函数，且最后一次为准。子线程也可更改信号处理函数，且随时都可改。

### 多线程下发送及接收信号的问题？

默认情况下只有主线程才可处理signal，就算指定子线程发送signal，也是主线程接收处理信号。

那么Golang如何做到给指定子线程发signal且处理的？如何指定给某个线程发送signal？ 在glibc下可以使用pthread_kill来给线程发signal，它底层调用的是SYS_tgkill系统调用。

![https://xiaorui.cc/wp-content/uploads/2020/02/Jietu20200214-013121_Fotor.jpg](https://xiaorui.cc/wp-content/uploads/2020/02/Jietu20200214-013121_Fotor.jpg)

```go
// xiaorui.cc

#include "pthread_impl.h"

int pthread_kill(pthread_t t, int sig)
{
	int r;
	__lock(t->killlock);
	r = t->dead ? ESRCH : -__syscall(SYS_tgkill, t->pid, t->tid, sig);
	__unlock(t->killlock);
	return r;
}
```

那么在go runtime/sys_linux_amd64.s里找到了SYS_tgkill的汇编实现。os_linux.go中signalM调用的就是tgkill的实现。

```go
// xiaorui.cc
#define SYS_tgkill      234

TEXT ·tgkill(SB),NOSPLIT,0
    MOVQ    tgid+0(FP), DI
    MOVQ    tid+8(FP), SI
    MOVQ    sig+16(FP), DX
    MOVLSYS_tgkill, AX
    SYSCALL
    RET
```

runtime 代码

```go
// xiaorui.cc

func tgkill(tgid, tid, sig int)

// signalM sends a signal to mp.
func signalM(mp *m, sig int) {
    tgkill(getpid(), int(mp.procid), sig)
}
```

## 总结：

随着 golang 版本不断更新，runtime的功能越来越完善。现在看来基于信号的抢占式调度显得很精妙。下一篇文章继续写go1.4 defer的优化，简单说在多场景下编译器消除了deferproc压入和deferreturn插入调用，而是直接调用延迟方法。