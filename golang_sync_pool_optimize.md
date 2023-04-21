# golang 通过 lockfree 无锁优化 sync.pool 的锁竞争开销

sync.pool是golang的标准库，通过堆对象复用达到减少gc延迟的库。相比不断的创建堆对象，sync.pool通过对象复用确实可以减少gc的延迟。但sync.pool也是有损耗的，损耗主要体现在锁竞争上。go1.13版在sync.pool下了不少功夫来优化锁。

go sync.pool 优化提案，[https://github.com/golang/go/commit/d5fd2dd6a17a816b7dfd99d4df70a85f1bf0de31#diff-491b0013c82345bf6cfa937bd78b690d](https://github.com/golang/go/commit/d5fd2dd6a17a816b7dfd99d4df70a85f1bf0de31#diff-491b0013c82345bf6cfa937bd78b690d)

## 代码分析

这里简单说下sync.pool的逻辑，每个p都有独享的缓存队列，当g进行sync.pool操作时，先找到所属p的private，如果没有对象可用，加锁从 shared切片里获取数据。如果还没有拿到缓存对象，那么到其他P的poolLocal进行偷数据，如果偷不到，那么就创建新对象。

```go
# xiaorui.cc

type Pool struct {  
    local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal  
    localSize uintptr        // size of the local array  
   
    New func() interface{}  
}  
```

我们通过分析go 1.12 sync.pool的源码，可以发现sync.pool里会有各种的锁逻辑，从自己的shared拿数据加锁。getSlow()偷其他P缓存，也是需要给每个p加锁。put归还缓存的时候，还是会mutex加一次锁。

go mutex锁的实现原理，我在以前的文章中说过好几次了。简单说，他开始也是atomic cas自旋，默认是4次尝试，当还没有拿到锁的时候进行waitqueue gopack休眠调度处理，等待其他协程释放锁时进行goready调度唤醒。

race中的goroutine可以顺利的陷入wait queue里？不，lock_futex.go的futexsleep逻辑会加大你的锁竞争消耗。可以通过strace来追一下系统调用。

go在1.13的版本中优化了sync.pool的锁竞争问题，这里还改变了shared的数据结构，以前的版本用切片做缓存，现在换成了poolChain双端链表。这个双端链表的设计很有意思，你看sync.pool源代码会发现跟redis quicklist相似，都是链表加数组的设计。

![http://xiaorui.cc/wp-content/uploads/2019/06/1_BAH0gDeO2OuF-m2qwvQVpA.png](http://xiaorui.cc/wp-content/uploads/2019/06/1_BAH0gDeO2OuF-m2qwvQVpA.png)

```go
// xiaorui.cc

type poolChain struct {
   head *poolChainElt

   tail *poolChainElt
}

type poolChainElt struct {
   poolDequeue

   next, prev *poolChainElt
}

type poolDequeue struct {
   headTail uint64
   vals []eface
}

type eface struct {
   typ, val unsafe.Pointer
}
```

下面是go sync.pool get和put的锁优化实现，以前不管是获取本地的shared，还是偷其他p的shared，过程都需要加锁的。1.13 sync.pool是通过atomic优化了锁竞争。

```go
// xiaorui.cc

type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array
	New func() interface{}
}

// Local per-P Pool appendix.
type poolLocalInternal struct {
	private interface{} // Can be used only by the respective P.
	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}

type poolLocal struct {
	poolLocalInternal
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
        . . .
	l, _ := p.pin()  // 拿到g所属p的poolLocal, 并锁定

        // 相比1.12版本，取消了 mutex 调用.
	if l.private == nil {
		l.private = x
		x = nil
	}
	if x != nil {
                // 归还缓存
		l.shared.pushHead(x)
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}

func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
	l, pid := p.pin()
	x := l.private
	l.private = nil
	if x == nil {
		x, _ = l.shared.popHead()  // 从本地的shared拿缓存
		if x == nil {
			x = p.getSlow(pid) // 尝试偷缓存
		}
	}
	runtime_procUnpin()
	if x == nil && p.New != nil {
		x = p.New() // 没偷到缓存，先建缓存
	}
	return x
}

func (p *Pool) getSlow(pid int) interface{} {
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// Try to steal one element from other procs.
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}
        . . .
}

func (c *poolChain) popTail() (interface{}, bool) {
	d := loadPoolChainElt(&c.tail)
	if d == nil {
		return nil, false
	}
	for {
		d2 := loadPoolChainElt(&d.next)
		if val, ok := d.popTail(); ok {
			return val, ok
		}

		if d2 == nil {
			return nil, false
		}

                // 恩，使用atomic cas来替换tail
		if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&c.tail)), unsafe.Pointer(d), unsafe.Pointer(d2)) {
			storePoolChainElt(&d2.prev, nil)
		}
		d = d2
	}
}
```

### sync.pool怎么规避的死锁?

golang1.12版在private读取使用了runtime pin锁，本地shared队列使用了poolLocal的mutex锁，向其他p偷缓存时迭代每个p的shared队列，每次的迭代为了安全当然也会加锁。

那么可以想一下这样是否有死锁的风险？ 没有，我们可以从get方法跟下去，每个过程都会放锁，锁的粒度虽然很细，但释放及时。另外偷任务是顺序加锁的过程。不会出现互锁的死锁问题。

### atomic cas vs mutex

上面有说 golang的mutex的实现依赖atomic cas和runtime调度。经过测试在无竞争或者少竞争的情况下，mutex的调用耗时比atomic cas仅仅多几个ns。通常来说mutex适合大锁的场景，atomic cas适合小锁的场景。sync.pool的缓存的操作本就是很快，属于小锁的范围。

如果用atomic应用到放锁 “慢” 的场景，必然会造成忙轮询。atomic不是廉价的系统调用，单次调用约7ns左右，在密集竞争下忙轮询的最高时延干到几百+ns，当然你可以使用runtime.Gosched()来切调度，但又增加了runtime的调度成本。

这是我以前写过的mutex vs atomic的性能报告，有兴趣的可以看看。

- [https://github.com/rfyiamcool/benchmark_lock_performance](https://github.com/rfyiamcool/benchmark_lock_performance)
- [https://github.com/rfyiamcool/benchmark_atomic_performance](https://github.com/rfyiamcool/benchmark_atomic_performance)

### go sync.pool 为啥用atomic替换mutex？

上面说了atomic cas vs mutex的区别。结合sync.pool的场景来说，从shared获取缓存对象，这个操作本应是很快的。但如果用mutex，协程竞争下会被陷入到wait queue里，等待他人放锁后被runtime goready唤醒，推到runq，然后再被调度。而使用atomic cas就简单多了，大概率多轮几次就差不多能拿锁了。

## 总结:

sync.pool是优化gc的利器，但也不是多多益善，还是需要用测试的数据来说话。

我的一个小经验，只要看到pprof off-cpu火焰图和strace pselect6异常，那么可以考虑优化锁竞争了。
另外文章里我省略了 victim gc的优化，毕竟在1.12版本里已经有了 victim的gc优化。