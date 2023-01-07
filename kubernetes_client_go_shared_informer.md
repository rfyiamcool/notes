# 深入源码分析 kubernetes client-go sharedIndexInformer 和 SharedInformerFactory 的实现原理

本文基于 kubernetes/client-go 的 `v0.26.0` 版本进行分析.

#### 衔接上篇文章 ( k8s informer 机制的实现原理)

在阅读本文前, 建议先移步阅读上篇文章, 篇幅问题拆分成上下两篇.

上篇文章内容:

- reflector 反射器实现原理
- deltaFIFO 增量队列实现原理
- storeIndex 倒排索引的实现原理
- threadSafeMap 缓存的实现原理
- controller 控制的实现原理
- list/watch 拉取监听原理
- informer 整合所有组件实现 informer

[深入源码分析 kubernetes client-go list-watch 和 informer 机制的实现原理](https://github.com/rfyiamcool/notes/blob/main/kubernetes_client_go_informer.md)

## sharedIndexInformer

`sharedIndexInformer` 相比普通的 informer 来说, 可以共享 reflector 反射器, 业务代码可以注册多个 resourceEventHandler 方法, 无需重复创建 informer 做监听及事件注册. 

如果相同资源实例化多个 informer, 那么每个 informer 都有一个 reflector 和 store. 不仅会有数据序列化的开销, 而且缓存 store 不能复用, 可能一个对象存在多个 informer 的 store 里. 

下面 `sharedIndexInformer` 简化的实现原理架构图.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301061840196.png)

### 实例化 SharedIndexInformer 对象.

```go
func NewSharedIndexInformer(lw ListerWatcher, exampleObject runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
	return NewSharedIndexInformerWithOptions(
		lw,
		exampleObject,
		SharedIndexInformerOptions{
			ResyncPeriod: defaultEventHandlerResyncPeriod,
			Indexers:     indexers,
		},
	)
}

func NewSharedIndexInformerWithOptions(lw ListerWatcher, exampleObject runtime.Object, options SharedIndexInformerOptions) SharedIndexInformer {
	realClock := &clock.RealClock{}

	return &sharedIndexInformer{
		indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, options.Indexers),
		processor:                       &sharedProcessor{clock: realClock},
		listerWatcher:                   lw,
		...
	}
}
```

### 启动 sharedIndexInformer

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	// 实例化 deltaFIFO 队列
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KnownObjects:          s.indexer,
		EmitDeltaTypeReplaced: true,
	})

	// 实例化 informer 其他组件需求的 config 配置集
	cfg := &Config{
		Queue:             fifo,

		// reflector 内部会依赖这个做 list/watch 操作
		ListerWatcher:     s.listerWatcher,

		// 这个是 reflector 消费 deltafifo 触发的回调方法
		Process:           s.HandleDeltas,
	}

	func() {
		// 哎, 真是脱裤子放屁呀. 
		// 难道这里的匿名函数只是为了锁的 defer 退出调用?
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

		// 实例化 controller 对象
		s.controller = New(cfg)
	}()

	// 缓存变更检测, 不用纠结其实现
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)

	
	// 启动 sharedProcessor
	wg.StartWithChannel(processorStopCh, s.processor.run)

	// 启动 controller
	s.controller.Run(stopCh)
}
```

### 启动 sharedProcessor

启动 sharedProcessor 处理器, 遍历所有的 listeners 监听器, 每个 listener 启动两个协程处理 run 和 pop 方法.

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()

		// 遍历所有 listeners 的 run 和 pop 方法
		for listener := range p.listeners {
			p.wg.Start(listener.run) 
			p.wg.Start(listener.pop)
		}
	}()

	// 收到退出信号
	<-stopCh

	// 退出收尾操作, 遍历 listeners 执行退出.
	for listener := range p.listeners {
		close(listener.addCh)
	}

	// 等待所有关联的协程退出.
	p.wg.Wait()
}
```

### 添加 eventHandler

`sharedIndexInformer` 是支持动态添加 `ResourceEventHandler` 事件方法.

根据传入的 handler 对象构建 listener 监听器. 然后把监听器加到 listeners 数组里, 并启动 run 和 pop 两个协程.

```go
func (s *sharedIndexInformer) AddEventHandler(handler ResourceEventHandler) (ResourceEventHandlerRegistration, error) {
	return s.AddEventHandlerWithResyncPeriod(handler, s.defaultEventHandlerResyncPeriod)
}

const minimumResyncPeriod = 1 * time.Second

func (s *sharedIndexInformer) AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) (ResourceEventHandlerRegistration, error) {
	if resyncPeriod > 0 {
		// 最少一秒
		if resyncPeriod < minimumResyncPeriod {
			resyncPeriod = minimumResyncPeriod
		}
	}

	// 实例化一个 listener 监听对象
	listener := newProcessListener(handler, resyncPeriod, determineResyncPeriod(resyncPeriod, s.resyncCheckPeriod), s.clock.Now(), initialBufferSize, s.HasSynced)

	// 把构建的 listener 放到 processor 的 listeners 数组里,并启动两个协程处理 run 和 pop 方法.
	if !s.started {
		return s.processor.addListener(listener), nil
	}

	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// 同上
	handle := s.processor.addListener(listener)

	// 增加通知
	for _, item := range s.indexer.List() {
		listener.add(addNotification{newObj: item, isInInitialList: true})
	}
	return handle, nil
}
```

### HandleDeltas 核心处理函数

`HandleDeltas` 用来处理从 `DeltaFIFO` 拿到的 deltas 事件列表, 然后通知给所有的 lisenter 去处理.

```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}, isInInitialList bool) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	if deltas, ok := obj.(Deltas); ok {
		// processDeltas 是在 controller 里实现的.
		return processDeltas(s, s.indexer, s.transform, deltas, isInInitialList)
	}
	return errors.New("object given as Process argument is not Deltas")
}
```

虽然前面有分析过, 但为了避免大家切换上下文, 这里再简单说下他的实现. 

- 当 delta 事件类型为 `sync/replaced/added/updated` 时, 判断在 store 是否存在该对象, 存在则执行 store.update 操作和 OnUpdate 回调, 反之则执行 store.add 和 OnAdd 回调.
- 当 delta 事件类型为 deleted 时, 则执行 store.delete 方法. 执行注册方法的 OnDelete.

在 sharedIndexInformer 设计里, 这里的 store 是 indexer 索引存储, handler 则是 sharedIndexInformer 自身实现的 ResourceEventHandler 接口.

```go
func processDeltas(
	handler ResourceEventHandler,
	clientState Store,
	transformer TransformFunc,
	deltas Deltas,
	isInInitialList bool,
) error {
	// from oldest to newest
	for _, d := range deltas {
		...

		switch d.Type {
		case Sync, Replaced, Added, Updated:
			if old, exists, err := clientState.Get(obj); err == nil && exists {
				if err := clientState.Update(obj); err != nil {
					return err
				}
				handler.OnUpdate(old, obj)
			} else {
				if err := clientState.Add(obj); err != nil {
					return err
				}
				handler.OnAdd(obj, isInInitialList)
			}
		case Deleted:
			if err := clientState.Delete(obj); err != nil {
				return err
			}
			handler.OnDelete(obj)
		}
	}
	return nil
}
```

#### sharedIndexInformer 内的 evnetHandler 的实现

其实就是调用 processor.distribute 实现的, 没有直接使用 delta 结构, 而是使用 `Notification` 结构体封装了下.

```go
func (s *sharedIndexInformer) OnAdd(obj interface{}, isInInitialList bool) {
	s.cacheMutationDetector.AddObject(obj)
	// 添加, 无需同步
	s.processor.distribute(addNotification{newObj: obj, isInInitialList: isInInitialList}, false)
}

func (s *sharedIndexInformer) OnUpdate(old, new interface{}) {
	isSync := false
	...

	// 如果新旧的 resource version 相等时, 则需要 resync 同步.
	if accessor, err := meta.Accessor(new); err == nil {
		if oldAccessor, err := meta.Accessor(old); err == nil {
			isSync = accessor.GetResourceVersion() == oldAccessor.GetResourceVersion()
		}
	}

	s.cacheMutationDetector.AddObject(new)
	// 更新
	s.processor.distribute(updateNotification{oldObj: old, newObj: new}, isSync)
}

func (s *sharedIndexInformer) OnDelete(old interface{}) {
	// 删除, 无需同步
	s.processor.distribute(deleteNotification{oldObj: old}, false)
}
```

#### distribute 把事件通知给所有 listeners

`distribute` 收到变更的事件后, 遍历通知给所有的 listener 监听器, 这里的通知是把事件写到 listener 的 addCh 通道.

```go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	// 加锁, 遍历所有的 listeners 集合, 并未每个 listener 添加事件.
	for listener, isSyncing := range p.listeners {
		switch {
		case !sync:
			listener.add(obj)
		case isSyncing:
			listener.add(obj)
		default:
		}
	}
}

func (p *processorListener) add(notification interface{}) {
	// 把 obj 写到 listener 对应的 addCh 管道里.
	p.addCh <- notification
}
```

#### processorListener 消费

前面有说在添加 listener 监听器时, 启动两个协程去执行 pop() 和 run().

- `pop()` 监听 addCh 队列把 notification 对象扔到 nextCh 管道里. 
- `run()` 对 nextCh 进行监听, 然后根据不同类型调用不同的 `ResourceEventHandler` 方法. 

让人值得注意的是 `pop()` 方法, 该方法里对数据的搬运有些绕. 你需要细心推敲下代码, 其过程也只是 `addCh -> pendingNotifications -> nextCh` 而已.

addCh 和 nextCh 都是无缓冲的管道, 在这里只是用来做通知, 因为不好预设 channel 应该配置多大, 大了浪费, 小了则出现概率阻塞. 

那么如果解决 channel 不定长的问题? k8s 里很多组件都是使用一个 ringbuffer 来实现的元素暂存.

```go
func (p *processorListener) pop() {
	var nextCh chan<- interface{}
	var notification interface{}

	for {
		select {
		// 把从 addCh 获取的对象扔到 nextCh 里
		case nextCh <- notification:
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()

			// ok = false, ringbuffer 没有需要处理的
			if !ok {
				// 既然没有事情要做, 那么就设 nextCh 为 nil.
				// 当 nextCh 为 nil 时, select 忽略该 case.
				nextCh = nil
			}

		// 从 addCh 获取对象, 如果上一次的 noti 还未扔到 nextCh 里, 那么之后的对象扔到 buffer 里
		case notificationToAdd, ok := <-p.addCh:
			...

			// 当 notification 为空时, 给 nextCh 一个能用的 channel.
			if notification == nil {
				notification = notificationToAdd
				nextCh = p.nextCh
			} else {
				// 如果不为空, 则扔到 ringbuffer 缓冲里
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}

func (p *processorListener) run() {
	stopCh := make(chan struct{})
	wait.Until(func() {
		for next := range p.nextCh {
			switch notification := next.(type) {
			case updateNotification:
				// 调用已注册事件方法的 OnUpdate 方法
				p.handler.OnUpdate(notification.oldObj, notification.newObj)
			case addNotification:
				// 调用已注册事件方法的 OnAdd 方法
				p.handler.OnAdd(notification.newObj, notification.isInInitialList)
				...
			case deleteNotification:
				// 调用已注册事件方法的 OnDelete 方法
				p.handler.OnDelete(notification.oldObj)
			}
		}
		close(stopCh)
	}, 1*time.Second, stopCh)
}
```

### WaitForCacheSync

同步等待 informer 完成数据同步, 直到同步完成才退出, 不然一直调用 `informer.HasSynced` 判断是否完成同步.

```go
func WaitForCacheSync(stopCh <-chan struct{}, cacheSyncs ...InformerSynced) bool {
	err := wait.PollImmediateUntil(syncedPollPeriod,
		func() (bool, error) {
			for _, syncFunc := range cacheSyncs {
				if !syncFunc() {
					return false, nil
				}
			}
			return true, nil
		},
		stopCh)
	if err != nil {
		return false
	}

	return true
}
```

## SharedInformerFactory 的实现原理

`sharedIndexInformer` 相比单纯的 informer 来说, 允许多个调用方注册使用自定义的 ResourceEventHandler 构建的 listener 监听器, 当有事件到来时, 通知给所有的 listener, 其优点因为共用一个 reflector 组件而节省资源, 毕竟 reflector 会发起 list/watch 去监听 apiserver. 

`SharedInformerFactory` 工厂相比 `SharedIndexInformer` 来说, 组合了多个 informer 对象. 在一个 `SharedInformerFactory` 工厂对象里可以放不同类型的 sharedInformer 对象, 每个资源类型有单独的一个 sharedIndexInformer, 相同资源类型的使用同一个 informer 对象即可.

> 在 k8s 内置的组件中都使用 `NewSharedInformerFactory` 来处理事件.

### 使用例子

下面是使用 SharedIndexInformer 的 demo 例子.

```go
func main() {
	...

	// 实例化 informers 集合对象
	informers := informers.NewSharedInformerFactory(client, 0)

	// 获取 pod informer
	podInformer := informers.Core().V1().Pods().Informer()

	// 为 podInformer 注册 eventHandler
	podInformer.AddEventHandler(&cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			pod := obj.(*v1.Pod)
			t.Logf("pod added: %s/%s", pod.Namespace, pod.Name)
		},
	})

	// 启动 informers
	informers.Start(ctx.Done())

	// 等待同步数据到本地缓存
	cache.WaitForCacheSync(ctx.Done(), podInformer.HasSynced)
}
```

### 获取 sharedIndexInformer

篇幅原因, 这里拿 pods 资源举例说明.

上面的 demo 中使用 `informers.Core().V1().Pods().Informer()` 可以获取 pods 资源的 informer. 

其内部调用 factory 的 `InformerFor` 来寻找各个资源类型的 `cache.SharedIndexInformer`. 有则直接返回, 但如果先前没有创建过, 那么就需要调用 `NewFilteredPodInformer` 创建 `SharedIndexInformer` 共享 Informer 了. 

源码位置: `informers/factory.go`

```go
// 查找会创建 sharedInformer
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()

	// 获取资源类型
	informerType := reflect.TypeOf(obj)

	// 如果已创建过则返回
	informer, exists := f.informers[informerType]
	if exists {
		return informer
	}

	// 配置 resyncPeriod 时长
	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}

	// 创建一个 sharedIndexInformer 对象, 其实就是调用 NewFilteredXXXInformer 方法.
	// xxx 为各个资源类型的名字, 函数代码在 informers/v1 的各个文件里.
	informer = newFunc(f.client, resyncPeriod)

	// 赋值
	f.informers[informerType] = informer

	return informer
}
```

源码位置: `informers/core/v1/pod.go`

```go
func (f *podInformer) Informer() cache.SharedIndexInformer {
	return f.factory.InformerFor(&corev1.Pod{}, f.defaultInformer)
}

func (f *podInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
	// 这里注册了一个 indexFunc 索引函数, 使用 namespace 做为 indexer 的索引. 
	return NewFilteredPodInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
}

func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	// 创建 sharedIndexInformer 对象
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			// 注册 listFunc 方法
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(context.TODO(), options)
			},
			// 注册 watchFunc 方法
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
			},
		},
		&corev1.Pod{}, // 定义资源类型
		resyncPeriod,  // 重新同步时长 
		indexers,      // 定义索引函数
	)
}
```

### 启动 sharedInformerFactory

启动当前 informers 里的所有 informer.

```go
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()


	for informerType, informer := range f.informers {
		if !f.startedInformers[informerType] {
			f.wg.Add(1)
			informer := informer
			go func() {
				defer f.wg.Done()
				informer.Run(stopCh)
			}()
			f.startedInformers[informerType] = true
		}
	}
}
```

### 同步等待缓存同步

遍历当前的 informers 集合, 依次调用 `WaitForCacheSync` 来等待缓存同步完成.

```go
func (f *sharedInformerFactory) WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool {
	// 加锁获取 informers map 集合, 因为后面同步操作是阻塞的, 不能一直把持锁, 不加锁又会 data race.
	// 索性 copy 一份出来.
	informers := func() map[reflect.Type]cache.SharedIndexInformer {
		f.lock.Lock()
		defer f.lock.Unlock()

		informers := map[reflect.Type]cache.SharedIndexInformer{}
		for informerType, informer := range f.informers {
			if f.startedInformers[informerType] {
				informers[informerType] = informer
			}
		}
		return informers
	}()

	res := map[reflect.Type]bool{}
	for informType, informer := range informers {
		// 等待每个资源完成同步本地缓存.
		res[informType] = cache.WaitForCacheSync(stopCh, informer.HasSynced)
	}
	return res
}
```

### informer Lister 的实现原理

使用 `SharedInformerFactory` 不仅可以拿到共享的 informer 实例, 也可以拿到 lister 实例. 

拿下面的 pod 例子举例说明. 通过 lister 实例调用 `Pods()` 获取某个 namespace 下的所有 pods 对象, 通过 `List(labels.Selector)` 可以拿到符合 labels 条件的 pods 集合. 

```go
func main() {
	// 实例化 informers 集合对象
	informers := informers.NewSharedInformerFactory(client, 0)

	// 获取 pod informer
	podInformer := informers.Core().V1().Pods().Informer()

	// 获取 pod lister 实例
	podLister := informers.Core().V1().Pods().Lister()

	// 从缓存中获取 pods
	podLister.List(labels.Everything())
}
```

#### 创建 podLister 查询对象

传参 informer indexer 存储创建 podLister 对象. 这里的 indexer 底层就是 threadSafeMap.

代码位置: `informers/core/v1/pod.go`

```go
func (f *podInformer) Lister() v1.PodLister {
	return v1.NewPodLister(f.Informer().GetIndexer())
}

// 实例化 podLister 实例
func NewPodLister(indexer cache.Indexer) PodLister {
	return &podLister{indexer: indexer}
}
```

#### 遍历查找

`Lister` 的实现其实没那么高级, 使用 `cache.ListAll` 遍历 indexer 缓冲的所有对象, 然后挑出符合 labels 条件的 pods 对象. 虽然是 `List()` 是遍历查找的过程, 在本地会产生一点计算压力, 但节省了 apiserver 端的开销, 也减少了因网络访问带来的时延 latency. 

如果自定义的 `k8s operator` 有频繁的 labels 条件查询, 可以增加自定义的索引方法 `indexFunc` 构建倒排索引. 或者通过自定义的 ResourceEventHandler 自定义更好的倒排索引, 毕竟 store.Indexer 模式的索引键是个string, 复杂多重条件下表现力差些意思.

```go
type podLister struct {
	// 内置 indexer store 对象
	indexer cache.Indexer
}

func (s *podLister) List(selector labels.Selector) (ret []*v1.Pod, err error) {
	// 传递 indexer, selector, 回调方法
	err = cache.ListAll(s.indexer, selector, func(m interface{}) {
		// 把符合条件的 pod 放到 ret 里
		ret = append(ret, m.(*v1.Pod))
	})
	return ret, err
}

func (s *podLister) Pods(namespace string) PodNamespaceLister {
	// 先从 store.index 里获取获取相关的索引对应的 names,
	// 再从 threadSafeMap 的 items 缓存里通过 names 获取对象集合.
	return podNamespaceLister{indexer: s.indexer, namespace: namespace}
}
```

下面是 `ListAll()` 遍历匹配的实现过程.

代码位置: `tools/cache/listers.go`

```go
type AppendFunc func(interface{})

func ListAll(store Store, selector labels.Selector, appendFn AppendFunc) error {
	selectAll := selector.Empty()
	// store.List 的内部实现是加锁, 然后把所有的对象放到 slice 里返回.
	for _, m := range store.List() {
		// 空的 selector
		if selectAll {
			appendFn(m)
			continue
		}

		metadata, err := meta.Accessor(m)
		// 满足匹配条件则回调添加
		if selector.Matches(labels.Set(metadata.GetLabels())) {
			appendFn(m)
		}
	}
	return nil
}
```

下面是 `store.List` 的实现.

代码位置: `tools/cache/thread_safe_store.go`

```go
func (c *threadSafeMap) List() []interface{} {
	c.lock.RLock()
	defer c.lock.RUnlock()

	list := make([]interface{}, 0, len(c.items))
	for _, item := range c.items {
		list = append(list, item)
	}
	return list
}
```

## 总结

到此 sharedIndexInformer 和 SharedInformerFactory 的实现过程分析完了, 不在详细的复述, 可通过下面的流程图加深其印象.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301061840196.png)