# 深入源码分析 kubernetes client-go list-watch 和 informer 机制的实现原理

本文基于 kubernetes/client-go 的 `v0.26.0` 版本进行分析.

**主要分析几个模块:**

1. reflector 反射器

通过 list/watch 监听 apiserver, 后面把增量的数据推到 deltaFIFO 增量事件队列里

2. deltaFIFO 增量队列

增量队列, 存储 delta 事件.

3. storeIndex 索引

index 就是存储了索引, 其目的就是为了加速数据的检索. 通过索引值只是拿到资源的 name, 获取对象还是存储在 `threadSafeMap` 里.

4. threadSafeMap 对象缓存

本地缓存, storeIndex 是索引, threadSafeMap 是存储资源对象的缓存.

5. controller 控制器

实例化并启动 reflector 反射器, 并调用 processLoop 来消费 deltaFIFO 队列.

6. informer

把上面的这几个模块组合起来就实现的 informer 的功能.

#### informer 架构图

为了更好的理解 informer 的设计, 所以画了一个简化的 informer 架构图 ( 忽略了各个组件内部的设计 ). 

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301062017062.png)

#### sharedIndexInformer 和 SharedInformerFactory 的实现原理

因本文篇幅略长, 所以把 `sharedIndexInformer` 和 `SharedInformerFactory` 的实现原理放到这里.

[深入源码分析 kubernetes client-go sharedIndexInformer 和 SharedInformerFactory 的实现原理](https://github.com/rfyiamcool/notes/blob/main/kubernetes_client_go_shared_informer.md)

## Reflector 的实现原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301060721747.png)

Reflector 的主要职责是从 k8s 的 apiserver 获取全量及监听增量事件, 把获取到的相关资源类型的增删改 `Add/Update/Delete` 事件写到 DeltaFIFO 递增队列里.

### 结构体

下面是 `reflector` 结构体.

```go
type Reflector struct {
	// 这个 store 指的是 deltaFIFO 队列.
	store Store

	// 实现了资源的 list 和 watch 接口.
	listerWatcher ListerWatcher

	// 从 apiserver 拉取到的最新的修订版号
	lastSyncResourceVersion
}
```

### 启动 reflector

启动 ListAndWatch 监听, 一直循环调用直到 stopCh 通知退出.

```go
func (r *Reflector) Run(stopCh <-chan struct{}) {
	wait.BackoffUntil(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			r.watchErrorHandler(r, err)
		}
	}, r.backoffManager, true, stopCh)
}
```

### 监听处理 listAndWatch

首先调用 `list()` 尝试获取某资源相关条件下的所有对象, 并记录当前最新的 `resourceVersion` 版本. 启动一个协程去处理 `resync` 定时同步逻辑, 默认不开启 resync 的, 也没必要开启该功能. 接着通过 `resourceVersion` 和 `timeoutSeconds` 参数实例化一个 watcher 对象, 该 watcher 会去监听 apiserver 提供的 watch 接口, 并把获取的事件往一个 resultChan 里输出. 最后通过 `watchHandler` 方法监听 `watcher.resultChan` 管道, 把拿到的事件扔到 `DeltaFIFO` 队列里.

```go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	// 首先尝试获取某资源相关条件下的所有对象
	err := r.list(stopCh)
	if err != nil {
		return err
	}

	resyncerrc := make(chan error, 1)
	cancelCh := make(chan struct{})
	defer close(cancelCh)
	go func() {
		// 通常 rsyncPeriod 为 0 , 不会触犯 resync 操作
		resyncCh, cleanup := r.resyncChan()
		defer func() {
			// 退出时关闭 resync 定时器
			cleanup() // Call the last one written into cleanup
		}()
		for {
			select {
			case <-resyncCh: // 触发 resync 定时器
				...
			}

			// 符合条件时进行重新同步.
			if r.ShouldResync == nil || r.ShouldResync() {
				if err := r.store.Resync(); err != nil {
					return
				}
			}

			// 退出时关闭 resync 定时器
			cleanup()
			resyncCh, cleanup = r.resyncChan()
		}
	}()

	// 配置带 backoff 退避的 rety 重试对象.
	retry := NewRetryWithDeadline(r.MaxInternalErrorRetryDuration, time.Minute, apierrors.IsInternalError, r.clock)
	for {
		...

		// 定义 http 流传输的超时时间
		timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
		options := metav1.ListOptions{
			// 上次的 resource version, 这样订阅到 apiserver 后, 可以拿到增量的数据.
			ResourceVersion: r.LastSyncResourceVersion(),
			// 配置超时时间.
			TimeoutSeconds: &timeoutSeconds,
		}

		// 创建一个 watcher 监听对象, 监听 apiserver 获取变更事件, 把新增事件扔到 watch.ResultChan 队列中.
		w, err := r.listerWatcher.Watch(options)
		if err != nil {
			return err
		}

		// 调用 `watcherHandler` 监听新增的事件, 然后把新增加到 DeltaFIFO 增量队列里.
		err = watchHandler(start, w, r.store, r.expectedType, r.expectedGVK, r.name, r.typeDescription, r.setLastSyncResourceVersion, r.clock, resyncerrc, stopCh)
		retry.After(err)
		if err != nil {
			if err != errorStopRequested {
				switch {
				case isExpiredError(err):
					...
				case apierrors.IsTooManyRequests(err):
					// 如果返回的错误是 http code 429, 则等待退避时间.
					<-r.initConnBackoffManager.Backoff().C()
					continue
				default:
					...
				}
			}
			return nil
		}
	}
}
```

####  list 拉取全量

`list-watch` 中的 `list()` 并不是每次都拉取全量的数据. 第一次拉取时由于 `resourceVersion` 为空, 所以拉取的是全量数据. 当 `list-watch` 出现异常进行重试重连时, `list()` 拉取的 resourceVersion 为上次最新的版本, 这样 list 会获取比该版本更新的所有数据.

```go
func (r *Reflector) list(stopCh <-chan struct{}) error {
	var resourceVersion string

	// 创建一个含有上次的 resourceVersion 版本的 options
	options := metav1.ListOptions{ResourceVersion: r.relistResourceVersion()}

	var list runtime.Object
	var paginatedResult bool

	go func() {
		// 使用 tool/pager 组装分页逻辑
		pager := pager.New(pager.SimplePageFunc(func(opts metav1.ListOptions) (runtime.Object, error) {
			return r.listerWatcher.List(opts)
		}))

		// 调用 pager.List 获取数据
		list, paginatedResult, err = pager.List(context.Background(), options)
		// 如果过期或者不合法 resourceversion 则进行重试.
		if isExpiredError(err) || isTooLargeResourceVersionError(err) {
			r.setIsLastSyncResourceVersionUnavailable(true)
			list, paginatedResult, err = pager.List(context.Background(), metav1.ListOptions{ResourceVersion: r.relistResourceVersion()})
		}
		close(listCh)
	}()

	if options.ResourceVersion == "0" && paginatedResult {
		r.paginatedResult = true
	}

	// 获取当前最新的版本
	listMetaInterface, err := meta.ListAccessor(list)
	resourceVersion = listMetaInterface.GetResourceVersion()

	// 转换数据结构
	items, err := meta.ExtractList(list)

	// 这里很关键, 把 items 数据同步到 store 里.
	if err := r.syncWith(items, resourceVersion); err != nil {
		return fmt.Errorf("unable to sync list result: %v", err)
	}

	// 更新 resourceVersion 
	r.setLastSyncResourceVersion(resourceVersion)
	return nil
}

func (r *Reflector) syncWith(items []runtime.Object, resourceVersion string) error {
	found := make([]interface{}, 0, len(items))
	for _, item := range items {
		found = append(found, item)
	}

	// 使用 store replace 写到队列中, 其过程还是较为复杂的.
	return r.store.Replace(found, resourceVersion)
}
```

####  watcherHandler

`watcherHandler` 的逻辑是从 watcher 的 ResultChan 管道中获取变更事件, 然后添加到 DeltaFIFO 队列中. store 的 `Add/Update/Delete` 操作其实在 DeltaFIFO 里都是插入的逻辑, 只是插入的事件类型为 `Add/Update/Delete` 里的一个.

```go
// watchHandler watches w and sets setLastSyncResourceVersion
func watchHandler(start time.Time,
	w watch.Interface,
	store Store,
	...
) error {

	eventCount := 0

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
			if !ok {
				// 退出 loop 
				break loop
			}
			if event.Type == watch.Error {
				return apierrors.FromObject(event.Object)
			}

			...

			meta, err := meta.Accessor(event.Object)
			if err != nil {
				continue
			}

			// 获取当前对象的 resourceVersion
			resourceVersion := meta.GetResourceVersion()

			switch event.Type {
			case watch.Added:
				// 新增事件
				err := store.Add(event.Object)

			case watch.Modified:
				// 更新事件
				err := store.Update(event.Object)

			case watch.Deleted:
				// 删除事件
				err := store.Delete(event.Object)
			case watch.Bookmark:
				// A `Bookmark` means watch has synced here, just update the resourceVersion
			default:
			}

			// 更新 resource version 版本, 下次使用该 resourceVersion 来 watch 监听. 
			setLastSyncResourceVersion(resourceVersion)
			if rvu, ok := store.(ResourceVersionUpdater); ok {
				rvu.UpdateResourceVersion(resourceVersion)
			}
			eventCount++
		}
	}

	watchDuration := clock.Since(start)
	if watchDuration < 1*time.Second && eventCount == 0 {
		// 如果 watch 退出小于 一秒, 另外一条事件也没拿到, 则打条错误日志
		return fmt.Errorf("very short watch: %s: Unexpected watch close - watch lasted less than a second and no items received", name)
	}
	...
	return nil
}
```

### informer watch 的实现原理

首先当客户端通过 watch api 监听 apiserver 后, apiserver 通过在返回 http header 中配置 `Transfer-Encoding: chunked` 实现分块编码传输. apiserver 把需要传递的数据按照 chunked 块的方法流式写入.

而客户端收到 apiserver 经过 chunked 编码的数据流后, 同样按照 http chunked 的方式进行解码. 

一句话总结, informer watch 流式传输是通过 http chunked 块传输实现的, 后面会专门补一篇文章分析下 client-go watch 实现原理.

在 apiserver 中可以看到 watch 接口在返回数据使用了 chunked 分块传输.

```go
e := streaming.NewEncoder(framer, s.Encoder)

// ensure the connection times out
timeoutCh, cleanup := s.TimeoutFactory.TimeoutCh()
defer cleanup()

// begin the stream
w.Header().Set("Content-Type", s.MediaType)
w.Header().Set("Transfer-Encoding", "chunked")
w.WriteHeader(http.StatusOK)
flusher.Flush()
```

从客户端返回的数据类似这样.

```c
$ curl -i http://{kube-api-server-ip}:8080/api/v1/watch/pods?watch=yes
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 02 Jan 2020 20:22:59 GMT
Transfer-Encoding: chunked

{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"MODIFIED", "object":{"kind":"Pod","apiVersion":"v1",...}}
```

## DeltaFIFO 的实现原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/Jietu20230106-063659.jpg)

### 队列中的事件类型

DeltaFIFO 定了以下这些变更类型:

```go
// DeltaType is the type of a change (addition, deletion, etc)
type DeltaType string

// Change type definition
const (
	Added    DeltaType = "Added"
	Updated  DeltaType = "Updated"
	Deleted  DeltaType = "Deleted"
	Replaced DeltaType = "Replaced"
	Sync     DeltaType = "Sync"
)
```

### 结构体定义

下面是 `DeltaFIFO` 增量队列的数据结构的定义.

`queue` 用来存对象格式化后的 key, 按照 fifo 的定义先进先出, 写是往后面 queue 这个 slice 结构 append, 读取则获取首个元素 `queue, item = queue[1:], queue[0]`.

`items` 作为字典存储了 Deltas 数据, key 为 queue 的元素, val 为 Delta 列表.

```go
type DeltaFIFO struct {
	// 为了保证 items 和 queue 的并发下读写安全
	lock sync.RWMutex
	cond sync.Cond

	// `items` maps a key to a Deltas.
	items map[string]Deltas

	// `queue` maintains FIFO order of keys for consumption in Pop().
	queue []string

	...
}

type Delta struct {
	// 事件类型
	Type   DeltaType

	// k8s 中资源的对象, 比如 pod, service, deployment 等对象.
	Object interface{}
}

type Deltas []Delta
```

### 计算对象的 key

默认情况下 deltaFIFO 的 items 的 key 和 queue 的元素都是通过 `MetaNamespaceKeyFunc` 计算出来的. 

该函数可以从 k8s 任意资源对象提取 namespace 和 name, 然后格式为 key string. 当有 namespace 不为空时, 则使用 `namespace/name`, 为空则使用资源的 name 作为 key.

```go
func MetaNamespaceKeyFunc(obj interface{}) (string, error) {
	if key, ok := obj.(ExplicitKey); ok {
		return string(key), nil
	}
	meta, err := meta.Accessor(obj)
	if err != nil {
		return "", fmt.Errorf("object has no meta: %v", err)
	}
	if len(meta.GetNamespace()) > 0 {
		return meta.GetNamespace() + "/" + meta.GetName(), nil
	}
	return meta.GetName(), nil
}
```

### 添加元素

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301060639663.png)

```go
func (f *DeltaFIFO) Add(obj interface{}) error {
	// 加锁
	f.lock.Lock()
	defer f.lock.Unlock()

	f.populated = true
	return f.queueActionLocked(Added, obj)
}

func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
	// 通过 obj 拼凑 id, 格式为 namespace/name
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	
	从 items 获取已经存在 deltas 列表
	oldDeltas := f.items[id]

	// 把新增的事件加入到已存在的 deltas
	newDeltas := append(oldDeltas, Delta{actionType, obj})

	newDeltas = dedupDeltas(newDeltas)

	if len(newDeltas) > 0 {
		// 判断是否 items 是否存在
		if _, exists := f.items[id]; !exists {
			// 不存在说明, queue 里也是空的, 在 queue 里也加入这个 delta
			f.queue = append(f.queue, id)
		}

		// 在 items 字典里加入这个 id 对应的 deltas 数组
		f.items[id] = newDeltas

		// 唤醒其他协程
		f.cond.Broadcast()
	} else {
		// 如果 newDeltas 为空, 返回错误.
		f.items[id] = newDeltas
		return fmt.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; broke DeltaFIFO invariant by storing empty Deltas", id, oldDeltas, obj)
	}
	return nil
}
```

下面是 delta 的去重逻辑. 拿倒数第一个 delta 跟倒数第二个 delta 做对比. 如果两个都是 Deleted 类型, 则把两个删除类型的 delta 去重下, 最后保留一个 deleted 类型的 delta 对象.

```go
func dedupDeltas(deltas Deltas) Deltas {
	n := len(deltas)
	if n < 2 {
		return deltas
	}

	a := &deltas[n-1]
	b := &deltas[n-2]
	if out := isDup(a, b); out != nil {
		deltas[n-2] = *out  // 赋值
		return deltas[:n-1] // 缩减一个槽位
	}
	return deltas
}

func isDup(a, b *Delta) *Delta {
	if out := isDeletionDup(a, b); out != nil {
		return out
	}
	return nil
}

func isDeletionDup(a, b *Delta) *Delta {
	if b.Type != Deleted || a.Type != Deleted {
		return nil
	}
	if _, ok := b.Object.(DeletedFinalStateUnknown); ok {
		return a
	}
	return b
}
```

### 消费元素

```go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		// 如果 queue 队列为空, 则使用 cond.Wait 陷入等待.
		for len(f.queue) == 0 {
			if f.closed {
				return nil, ErrFIFOClosed
			}

			f.cond.Wait()
		}

		// 从队列头部获取元素
		id := f.queue[0]

		// 收缩队列去除头部
		f.queue = f.queue[1:]

		// 获取 deltas 对象
		item, ok := f.items[id]
		if !ok {
			// 不应该拿到, 二次判断
			continue
		}

		// 从 items 字典中删除
		delete(f.items, id)

		// 把上面获取的 deltas 对象交给 process 处理
		err := process(item, isInInitialList)
		if e, ok := err.(ErrRequeue); ok {
			// 如果需要重新入队, 则调用 addIfNotPresent 入队.
			// 该函数入队前会做重复判断, 不存在才插入.
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		
		// 返回 deltas 变更集合
		return item, err
	}
}
```

调用传入的 process 方法其实就是 controller 的 `HandleDeltas`, 内部调用了实例化 informer 时, 注册的 eventHandler 方法. 按照 deltaType 类型, 选择调用 `OnAdd OnUpdate OnDelete` .

### 去重插入

插入 DeltaFIFO 队列时, 判断是否已存在该元素, 只插入不存在的元素.

```go
func (f *DeltaFIFO) AddIfNotPresent(obj interface{}) error {
	deltas, ok := obj.(Deltas)
	if !ok {
		return fmt.Errorf("object must be of type deltas, but got: %#v", obj)
	}
	id, err := f.KeyOf(deltas)
	if err != nil {
		return KeyError{obj, err}
	}
	f.lock.Lock()
	defer f.lock.Unlock()
	f.addIfNotPresent(id, deltas)
	return nil
}

func (f *DeltaFIFO) addIfNotPresent(id string, deltas Deltas) {
	// 在 items 存在, 则跳出.
	if _, exists := f.items[id]; exists {
		return
	}

	// 加入到 queue 和 items 字典中.
	f.queue = append(f.queue, id)
	f.items[id] = deltas
	f.cond.Broadcast()
}
```

### DeltaFIFO 队列实现小结:

每次想 deltaFIFO 队列添加资源对象的 delta 事件时, 需要判断队列中是否存储过, 有则在关联的 Deltas 列表后面追加新的 delta. 队列中不存在, 则在 queue 和 items 加一遍.

需要重点说的一点, 添加已有的 key 时, 不会调整该 key 在 queue 的顺序. DeltaFIFO 增量队列的 FIFO 先进先出只是对同一个 key 的 Deltas 列表.

比如, 在 quque 里顺序添加 obkey1 ，再添加 obkey2. 当后面如果在添加一个  obkey1 的 updated 的事件, 消费者在读取时可以拿到 obkey1 的所有事件，包括后面的 Updated.

## Indexer 索引的实现原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/informer-indexer.png)

`storeIndex` 实现了 indexer 对象索引存储功能. 实例化 indexer 索引对象时, 注册计算索引的方法. 后面每次新增对象时会使用 indexFunc 计算出需要索引的值列表, 通过倒排的方式来组织写入. 读取的时候需要从指定 indexFunc 名字的 Index 里读取.

### 索引的数据结构

下面是 storeIndex 索引的数据结构. 

```go
type storeIndex struct {
	indexers Indexers

	indices Indices
}


// key 为 索引函数的名字, value 为 IndexFunc 类型的索引函数 
type Indexers map[string]IndexFunc

// key 为 索引函数的名字, value 是一个 Index 结构. 
// 相当于倒排的逻辑, 比如 annotation 里含有 nginx 字符串的有哪些 names.
type Indices map[string]Index

// key 为索引条件, value 为一个集群, 存储了符合条件的 names 集合.
type Index map[string]sets.String
```

### indexer 更新索引

`updateIndices` 方法是用来更新索引, 其内部逻辑是这样的, 遍历所有注册的 indexer 索引方法集合, 然后使用 `indexFunc` 计算出 oldobj 和 newobj 的索引值. 后面删除旧的 obj 的索引值, 接着添加新的 obj 索引值.

这里的 key 一般是资源对象的 `namespace/name` 值, indexer 使用这个 key 来描述具体的资源对象, 后面可以使用这个 key 在 threadSafeMap 里获取真正的 obj.

```go
func (i *storeIndex) updateIndices(oldObj interface{}, newObj interface{}, key string) {
	var oldIndexValues, indexValues []string
	var err error
	for name, indexFunc := range i.indexers {
		// 通过 indexFunc 方法计算 obj 的索引值列表
		if oldObj != nil {
			oldIndexValues, err = indexFunc(oldObj)
		} else {
			oldIndexValues = oldIndexValues[:0]
		}
		...

		// 通过 indexFunc 方法计算 obj 的索引值列表
		if newObj != nil {
			indexValues, err = indexFunc(newObj)
		} else {
			indexValues = indexValues[:0]
		}
		...

		index := i.indices[name]
		if index == nil {
			index = Index{}
			i.indices[name] = index
		}

		// 如果新旧对象的索引一样, 则无需变更.
		if len(indexValues) == 1 && len(oldIndexValues) == 1 && indexValues[0] == oldIndexValues[0] {
			continue
		}

		// 在 index 里删除旧的 obj 的索引值列表
		for _, value := range oldIndexValues {
			i.deleteKeyFromIndex(key, value, index)
		}

		// 在 index 里添加更新的 obj 的索引值列表
		for _, value := range indexValues {
			i.addKeyToIndex(key, value, index)
		}
	}
}

// 添加索引, 直接在 index 关联的 set 集合里添加 key.
func (i *storeIndex) addKeyToIndex(key, indexValue string, index Index) {
	set := index[indexValue]
	if set == nil {
		set = sets.String{}
		index[indexValue] = set
	}
	// 添加到 set 里.
	set.Insert(key)
}

// 删除索引, 直接在 index 关联的 set 集合里删除 key.
func (i *storeIndex) deleteKeyFromIndex(key, indexValue string, index Index) {
	set := index[indexValue]
	if set == nil {
		return
	}

	// 在 set 中删除
	set.Delete(key)
	if len(set) == 0 {
		// set 为空, 在 map 中剔除.
		delete(index, indexValue)
	}
}
```

### getKeysByIndex 读取索引

`getKeysByIndex` 方法用来读取索引, 调用时传入两个参数, indexName 为索引函数名， indexValue 为索引值.

源码逻辑很简单, 判断 indexName 是否注册过, 接着从 indeces 中获取 indexName 对应的 index 结构, 再返回 indexValue 对应的 names 集合.  

```go
func (i *storeIndex) getKeysByIndex(indexName, indexedValue string) (sets.String, error) {
	// 判断 indexName 是否注册过, 为空则不合法
	indexFunc := i.indexers[indexName]
	if indexFunc == nil {
		return nil, fmt.Errorf("Index with name %s does not exist", indexName)
	}

	// 获取 indexName 索引函数名对应的 Index 结构
	index := i.indices[indexName]
	// 从 Index 结构中获取跟 indexValue 相关的 names 集合.
	return index[indexedValue], nil
}
```

## threadSafeMap 缓存资源对象的原理

`threadSafeMap` 用来维护索引和缓存资源对象的, 索引是使用 `storeIndex` 实现的, 资源对象的缓存则使用 `map[string]interface{}` 字典实现. 

需要注意的是 storeIndex 索引类一般不会在外部直接使用, 而是封装在 `threadSafeMap` 安全存储里. `threadSafeMap` 方法内部实现了 Add/Update/Delete 方法, 这类修改操作时不仅会从缓存中操作对象, 而且会对索引进行增删改. 另外 ByIndex 实现了从 indexer 中获取匹配索引的 names, 然后从缓存中获取匹配 names 的资源对象.

threadSafeMap 的代码位置: `tools/cache/thread_safe_store.go`

### 结构体定义

```go
// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
	lock  sync.RWMutex

	// 存储资源对象
	items map[string]interface{}

	// 存储资源对象的查询索引
	index *storeIndex
}
```

### 方法实现

直接看代码.

```go
// 传入自定义索引方法 IndexFunc 和 索引的缓存对象来创建 ThreadSafeStore
func NewThreadSafeStore(indexers Indexers, indices Indices) ThreadSafeStore {
	return &threadSafeMap{
		items: map[string]interface{}{},
		index: &storeIndex{
			indexers: indexers, // 自定义的索引函数 indexFunc
			indices:  indices,  // 没搞懂, 把控的 indices 传入进来干嘛.
		},
	}
}

// 添加对象
func (c *threadSafeMap) Add(key string, obj interface{}) {
	c.Update(key, obj)
}

func (c *threadSafeMap) Update(key string, obj interface{}) {
	c.lock.Lock()
	defer c.lock.Unlock()

	// 读取旧对象
	oldObject := c.items[key]

	// 赋值新对象
	c.items[key] = obj

	// 在 index 索引里, 删除旧对象关联的索引, 建立新对象的索引关系.
	c.index.updateIndices(oldObject, obj, key)
}

func (c *threadSafeMap) Delete(key string) {
	c.lock.Lock()
	defer c.lock.Unlock()
	if obj, exists := c.items[key]; exists {
		// 删除该对象的索引
		c.index.updateIndices(obj, nil, key)

		// 删除该对象的缓存
		delete(c.items, key)
	}
}

func (c *threadSafeMap) Get(key string) (item interface{}, exists bool) {
	c.lock.RLock()
	defer c.lock.RUnlock()

	// 通过 name 来获取资源对象
	item, exists = c.items[key]
	return item, exists
}

// 通过索引值来获取资源对象
func (c *threadSafeMap) ByIndex(indexName, indexedValue string) ([]interface{}, error) {
	c.lock.RLock()
	defer c.lock.RUnlock()

	// 调用 indexer 的 getKeysByIndex 获取 names 集合
	set, err := c.index.getKeysByIndex(indexName, indexedValue)
	if err != nil {
		return nil, err
	}

	list := make([]interface{}, 0, set.Len())
	for key := range set {
		// 遍历 set 集合并从 items 缓存中获取资源对象
		list = append(list, c.items[key])
	}

	return list, nil
}

// 添加注册自定义的索引方法
func (c *threadSafeMap) AddIndexers(newIndexers Indexers) error {
	c.lock.Lock()
	defer c.lock.Unlock()

	// 对比更新 index 中索引函数集合, 只是新增的 indexer 加到表里.
	return c.index.addIndexers(newIndexers)
}

省略一些代码
...
...
```

### threadSafeMap 小结

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301062003821.png)

以缓存索引条件为 `labels.webserver = nginx` 的 pods 为例. 

当需要读取 `labels.webserver = nginx` 的 pods 对象时, 先通过 threadSafeMap 的 storeIndex 获取符合条件的 pods 名字集合, 然后在从 threadSafeMap 的 items 里获取 pod 对象后返回.

而写流程的话, 先判断是否需要更新, 无需更新则直接建立索引, 并缓存对象. 如果需要更新, 则需求先清理以前的缓存, 再重建新对象的缓存, 最后覆盖缓存对象.

## controller 控制器实现原理

Controller 作为中心的控制器, 连接了 Reflector / DeltaFIFO / Indexer / Store 组件. 其内部逻辑会实例化 reflector, 然后启动 reflector, 接着使用 processLoop 来从 deltaFIFO 队列中获取事件.

### controller 数据结构的定义

代码位置: `tools/cache/controller.go`

```go
type controller struct {
	config         Config
	reflector      *Reflector
	reflectorMutex sync.RWMutex
	clock          clock.Clock
}

type Config struct {
    // 其实就是 DeltaFIFO 实现
    Queue

    // 构造 Reflector 需要
    ListerWatcher

    // Pop 出来的 obj 处理函数
    Process ProcessFunc

    // 目标对象类型
    ObjectType runtime.Object

    // Watch 返回 err 的回调函数
    WatchErrorHandler WatchErrorHandler

    // Watch 分页大小
    WatchListPageSize int64
}
```

### 启动控制器

```go
func (c *controller) Run(stopCh <-chan struct{}) {
	go func() {
		// 只是为了关闭 queue, 这开协程是真随意呀.
		<-stopCh
		c.config.Queue.Close()
	}()

	// 实例化 reflector 反射器
	r := NewReflectorWithOptions(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		ReflectorOptions{
		},
	)

	// k8s 自己抽象了层 waitgroup
	var wg wait.Group

	// 调用 reflector 的 Run 方法, 其内部会做 list-watch 操作, 然后把数据推到 deltaqueue 里.
	wg.StartWithChannel(stopCh, r.Run)

	// processLoop 从 deltaqueue 消费对象, 并处理对象. 
	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
```

`processLoop` 的逻辑是不断从 deltaQueue 里消费事件, 然后 deltaQueue 内部每次 pop 完之前会调用传入的 process 方法. 该方法是由 `processDeltas` 来实现的.

```go
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```

### 核心处理函数 processDeltas

`processDeltas` 的逻辑是遍历一个资源对象的 deltas 事件列表. 判断该对象在 store 是否存在, 不存在则创建, 反之则更新. 另根据事件的类型, 调用对应的用户注册的 `ResourceEventHandler` 事件回调方法.

```go
func processDeltas(
	// Object which receives event notifications from the given deltas
	handler ResourceEventHandler,
	clientState Store,
	transformer TransformFunc,
	deltas Deltas,
	isInInitialList bool,
) error {
	// from oldest to newest
	for _, d := range deltas {
		obj := d.Object

		switch d.Type {
		case Sync, Replaced, Added, Updated: // delta type
			// 如该对象已存在, 则尝试更新
			if old, exists, err := clientState.Get(obj); err == nil && exists {
				if err := clientState.Update(obj); err != nil {
					return err
				}
				// 调用 eventHandler 的 OnUpdate 方法
				handler.OnUpdate(old, obj)
			} else {
				// 如该对象不存在, 则创建
				if err := clientState.Add(obj); err != nil {
					return err
				}

				// 调用 eventHandler 的 OnAdd 方法
				handler.OnAdd(obj, isInInitialList)
			}
		case Deleted:
			// 在 store 里删除对象
			if err := clientState.Delete(obj); err != nil {
				return err
			}

			// 调用注册的 resourceEventHandler.OnDelete 方法
			handler.OnDelete(obj)
		}
	}
	return nil
}
```

### HasSynced 判断同步完成

`HasSynced` 是用来查询是否已经同步完数据的方法, 通常配合 `cache.WaitForCacheSync` 使用. 虽然在 controller 封装了该方法, 其实实现是在 `DeltaFIFO` 中.

#### 如何判定 `HasSynced()` 就同步完成了, 其依据是什么 ?

首先 `list-watch` 的机制是先 list 拉全量, 再 watch 监听. list 往队列写的 delta 类型是 Replaced, 且会累计加 `initialPopulationCount` 计数. 

当 list 都完事了后, watch 会使用 `Add/Update/Delete/AddIfNotPresent` 写队列时, 这时候 populated 为 true. 并且当 controller.processLoop 从 deltaFIFO 消费数据, 消费次数达到 `initialPopulationCount` 时可判定同步完成.

**一句话总结:**

当 `list()` 把数据都推到 deltaqueue 里, 然后 `controller.processLoop` 消费完队列中 list() 产生的数据, 那么就认为同步完成了.

```go
func (c *controller) HasSynced() bool {
	return c.config.Queue.HasSynced()
}

func (f *DeltaFIFO) HasSynced() bool {
	f.lock.Lock()
	defer f.lock.Unlock()
	return f.hasSynced_locked()
}

func (f *DeltaFIFO) hasSynced_locked() bool {
	return f.populated && f.initialPopulationCount == 0
}
```

## Informer 的实现原理

到现在终于讲到 informer 了. 

其实单单 informer 的实现还是比较简单的, 内部依赖 controller 实现 informer 的功能.

### informer 的使用案例

分析 informer 源码之前先来看看 informer 是怎么用的, client-go 代码库中有个 example 目录, 其中是有 informer 使用的例子.

源码位置: `examples/workqueue/main.go`

```go
func main() {
	...

	// 实例化 apiserver 客户端
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatal(err)
	}

	// 实例化 pod 类型的 list watcher 对象
	podListWatcher := cache.NewListWatchFromClient(clientset.CoreV1().RESTClient(), "pods", v1.NamespaceDefault, fields.Everything())

	// 实例化支持 indexer 自定义索引方法的 informer
	indexer, informer := cache.NewIndexerInformer(podListWatcher, &v1.Pod{}, 0, cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			...
		},
		UpdateFunc: func(old interface{}, new interface{}) {
			...
		},
		DeleteFunc: func(obj interface{}) {
			...
		},
	}, cache.Indexers{})

	// 启动 informer
	go informer.Run(stopCh)

	// 等待同步数据到本地缓存
	if !cache.WaitForCacheSync(stopCh, c.informer.HasSynced) {
		return
	}
	...
}
```

### informer 的创建

informer 内部会实例化 indexer, delta fifo, controller 对象. 另外在 config 里准备好 reflector 启动所需的 listerWatcher / fifo / ... 对象.

```go
func NewIndexerInformer(
	lw ListerWatcher,
	objType runtime.Object,
	resyncPeriod time.Duration,
	h ResourceEventHandler,
	indexers Indexers,
) (Indexer, Controller) {

	// 实例化 indexer 存储对象
	clientState := NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers)

	return clientState, newInformer(lw, objType, resyncPeriod, h, clientState, nil)
}

func newInformer(
	lw ListerWatcher,
	h ResourceEventHandler,
	clientState Store,
	...
) Controller {

	// 实例化 DeltaFIFO 增量队列
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KnownObjects:          clientState,
		EmitDeltaTypeReplaced: true,
	})

	// 创建 config 集合对象, 内置 fifo, listerwatcher, process 对象.
	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    lw,
		ObjectType:       objType,
		...
		Process: func(obj interface{}, isInInitialList bool) error {
			if deltas, ok := obj.(Deltas); ok {
				return processDeltas(h, clientState, transformer, deltas, isInInitialList)
			}
			return errors.New("object given as Process argument is not Deltas")
		},
	}

	// 实例化 controller 对象, 注意 newinformer 返回的是 controller 方法
	return New(cfg)
}

// 不再复述
func New(c *Config) Controller {
	ctlr := &controller{
		config: *c,
	}
	return ctlr
}
```

### informer 的启动

调用 informer.Run() 其实是调用 controller.Run() 方法. 前面有详细讲过 controller 的流程.

简单说就是 `Run()` 内部会实例化 reflector 反射器对象, 并启动 reflector 的 List / Watch 监听 apiserver 事件. 当拿到 apiserver 的数据事件时, 把数据推到 deltaFIFO 队列中. 

**那么谁会消费 deltaFIFO 的数据 ?**

controller 还会启动一个 processLoop 方法, 其逻辑是从 deltaFIFO 队列中获取事件, 然后更新 store 和触发运行用户注册的 resourceEventHandler 事件回调方法.

```go
informer.Run(shopCh)
```

### WaitForCacheSync

`WaitForCacheSync` 用来等待 informer 同步完成. 其内部逻辑是每隔 `100ms` 调用传入的 `HasSynced()` 方法, 直到同步完成才退出. 

`cache.WaitForCacheSync(stopCh, c.informer.HasSynced)`

```go
const syncedPollPeriod = 100 * time.Millisecond

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

### informer 小结

单单由 NewIndexerInformer 创建的 informer 的实现是比较简单的, 它的内部依赖 controller 实现 informer 的功能. controller 又会关联 reflector, deltaFIFO, Store (indexer, threadSafeMap ) 组件之间的协调联动.

## 总结

client-go 的 informer 过程实现还是颇为复杂, 主要是其内部由多个组件构成, 组件之间需要控制器来协调. 详细的过程就不在复述了, 可通过下面的流程图加深其印象.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301062017062.png)