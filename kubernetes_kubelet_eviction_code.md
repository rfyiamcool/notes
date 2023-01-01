## 源码分析 kubelet eviction manager 驱逐的实现原理

kubernetes 出于保证节点的服务质量和稳定性，在节点资源紧张时会驱动 kubelet 来做 pods 驱逐. `kubelet eviction manager` 就是驱逐 pod 的具体实现, 该模块主要是检测当前 node 资源是否紧张, 在紧张时按照策略会驱逐 pod. 

`基于k8s v1.27.0 源码分析.`

### 启动入口

eviction manager 支持通过注册事件通知的机制来观测 cgroups 内存, 当达到阈值时调用 synchronize 同步方法. `KernelMemcgNotification` 默认为 false, 不主动开启的, 其实依赖于轮询检测也够用了.

```go
func (m *managerImpl) Start(diskInfoProvider DiskInfoProvider, podFunc ActivePodsFunc, podCleanedUpFunc PodCleanedUpFunc, monitoringInterval time.Duration) {
	thresholdHandler := func(message string) {
		// 核心方法
		m.synchronize(diskInfoProvider, podFunc)
	}

	// 如果开启内核的内存通知机制
	if m.config.KernelMemcgNotification {
		for _, threshold := range m.config.Thresholds {
			// 由于 cgroups 只实现了 memory 事件监测, 故而这里只匹配内存相关的阈值配置.
			if threshold.Signal == evictionapi.SignalMemoryAvailable || threshold.Signal == evictionapi.SignalAllocatableMemoryAvailable {
				notifier, err := NewMemoryThresholdNotifier(threshold, m.config.PodCgroupRoot, &CgroupNotifierFactory{}, thresholdHandler)
				if err != nil {
				} else {
					// 启动事件内存检测
					go notifier.Start()
					m.thresholdNotifiers = append(m.thresholdNotifiers, notifier)
				}
			}
		}
	}

	// 启用后台任务
	go func() {
		for {
			// synchronize 是 eviction manager 核心的同步方法
			if evictedPods := m.synchronize(diskInfoProvider, podFunc); evictedPods != nil {
				// 阻塞等待驱逐的 pods 都被清理.
				m.waitForPodsCleanup(podCleanedUpFunc, evictedPods)
			} else {
				// 每 10 秒进行一次 synchronize 同步.
				time.Sleep(monitoringInterval)
			}
		}
	}()
}
```

### cgroup memory 事件通知的实现

> 需要注意的是 cgroup 现在只实现了内存阈值的事件通知, 其他事件监测还未实现.

创建两个文件描述符 watchfd 和 controlfd, watchfd 路径为 `/sys/fs/cgroups/memory/memory.usage_in_bytes`, controlfd 路径为 `/sys/fs/cgroups/memory/cgroup.event_control`
接着创建一个 eventfd 事件文件描述符和 epoll, 往 `controlfd` 写一条配置数据, 声明关联的 eventfd, watchfd 和 threshold 阈值. 这样 cgroups 内部当监测到内存达到阈值后通知给 eventfd.

`start()` 方法里把 eventfd 注册到 epoll 里, 然后调用 epoll wait 进行监听, 当 eventfd 有事件到达时, 给 events 管道里发送事件, 通知上层代码进行驱逐.

代码位置: `pkg/kubelet/eviction/threshold_notifier_linux.go`

```go
func NewCgroupNotifier(path, attribute string, threshold int64) (CgroupNotifier, error) {
	var watchfd, eventfd, epfd, controlfd int
	var err error
	watchfd, err = unix.Open(fmt.Sprintf("%s/%s", path, attribute), unix.O_RDONLY|unix.O_CLOEXEC, 0)
	...

	controlfd, err = unix.Open(fmt.Sprintf("%s/cgroup.event_control", path), unix.O_WRONLY|unix.O_CLOEXEC, 0)
	...

	// 实例化 eventfd
	eventfd, err = unix.Eventfd(0, unix.EFD_CLOEXEC)
	if err != nil {
		return nil, err
	}

	// 创建 epoll
	epfd, err = unix.EpollCreate1(unix.EPOLL_CLOEXEC)
	if err != nil {
		return nil, err
	}

	// 构建一条配置, 关联 eventfd watchfd 和 内存阈值, 并通知给 cgroups
	config := fmt.Sprintf("%d %d %d", eventfd, watchfd, threshold)
	_, err = unix.Write(controlfd, []byte(config))
	if err != nil {
		return nil, err
	}
	return &linuxCgroupNotifier{
		eventfd: eventfd,
		epfd:    epfd,
		stop:    make(chan struct{}),
	}, nil
}

func (n *linuxCgroupNotifier) Start(eventCh chan<- struct{}) {
	// 把 eventfd 事件fd 加到 epoll 里做 epollin 读监听.
	err := unix.EpollCtl(n.epfd, unix.EPOLL_CTL_ADD, n.eventfd, &unix.EpollEvent{
		Fd:     int32(n.eventfd),
		Events: unix.EPOLLIN, // 监听 epollin 可读事件
	})
	...

	buf := make([]byte, eventSize)
	for {
		select {
		case <-n.stop:
			return
		default:
		}
		event, err := wait(n.epfd, n.eventfd, notifierRefreshInterval)
		...

		// 读取数据, 不关心内容, 只关心是否有事件到达.
		_, err = unix.Read(n.eventfd, buf)
		if err != nil {
			klog.InfoS("Eviction manager: error reading memcg events", "err", err)
			return
		}
		eventCh <- struct{}{}
	}
}

func wait(epfd, eventfd int, timeout time.Duration) (bool, error) {
	events := make([]unix.EpollEvent, numFdEvents+1)

	// 超时时间为 10 秒
	timeoutMS := int(timeout / time.Millisecond)

	// 调用 epoll wait 等待事件, 加入超时机制, 避免无事件到达时, 该系统调用无法退出的问题, 导致该 notifier 无法退出造成泄露.
	n, err := unix.EpollWait(epfd, events, timeoutMS)
	...
	for _, event := range events[:n] {
		if event.Fd == int32(eventfd) { // 只关心 eventfd
			if event.Events&unix.EPOLLHUP != 0 || event.Events&unix.EPOLLERR != 0 || event.Events&unix.EPOLLIN != 0 {
				return true, nil
			}
		}
	}

	return false, nil
}
```

events 管道是在哪里创建的? eviction manager 里会实例化 ThresholdNotifier 对象, 该对象里会创建 events 管道, 然后做监听 events 管道监听, 当有事件到达时就调用传递的 handler 方法, 这里的 handler 其实就是 驱逐的核心代码 `synchronize`.

```go
func NewMemoryThresholdNotifier(threshold evictionapi.Threshold, cgroupRoot string, factory NotifierFactory, handler func(string)) (ThresholdNotifier, error) {
	...
	return &memoryThresholdNotifier{
		threshold:  threshold,
		cgroupPath: cgpath,
		events:     make(chan struct{}), // 用来通知的 events 管道
		handler:    handler,
	}, nil
}

func (m *memoryThresholdNotifier) Start() {
	// 当 cgroups 有内置阈值事件到达时, 给 events 通知事件.
	for range m.events {
		m.handler(fmt.Sprintf("eviction manager: %s crossed", m.Description()))
	}
}

func (m *memoryThresholdNotifier) UpdateThreshold(summary *statsapi.Summary) error {
	...
	go m.notifier.Start(m.events)
	return nil
}
```

### synchronize 核心处理函数

synchronize 是 eviction manager 核心的处理方法. 操作流程如下:

1. 使用 `summaryProvider.Get` 获取当前节点的资源使用情况, 比如 cpu, memory, network 等等
2. 获取当前活跃的 pods
3. 查看当前哪些资源达到了被驱逐的阈值
4. 尝试进行 imageGC 和 containerGC 垃圾回收清理
5. 按照驱逐优先级对阈值类型进行排序
6. 再对 pods 进行优先级排序
7. 对排序后的 pods 进行驱逐,  清理一个就退出, 如节点资源依旧紧缺则下次清理

```go
func (m *managerImpl) synchronize(diskInfoProvider DiskInfoProvider, podFunc ActivePodsFunc) []*v1.Pod {
	if m.dedicatedImageFs == nil {
		hasImageFs, ok := diskInfoProvider.HasDedicatedImageFs(ctx)
		if ok != nil {
			return nil
		}

		// 把各个 signal 的 rankFunc 排序注册到一个map里, 后面对 pods 进行排序会用到这个 rank.
		m.signalToRankFunc = buildSignalToRankFunc(hasImageFs)

		// 把 imageGc, containerGC 注册到 node 的清理函数, 在驱逐 pods 前会调用一次.
		m.signalToNodeReclaimFuncs = buildSignalToNodeReclaimFuncs(m.imageGC, m.containerGC, hasImageFs)
	}

	// 获取激活的 pods 列表
	activePods := podFunc()

	// 获取当前节点的资源使用情况, 比如 cpu, memory, fs, network, inode 等等
	summary, err := m.summaryProvider.Get(ctx, updateStats)
	if err != nil {
		klog.ErrorS(err, "Eviction manager: failed to get summary stats")
		return nil
	}

	// 如果离上次操作超过10s, 调用 notifier 的 UpdateThreshold 接口, 该接口会启用阈值的监测
	if m.clock.Since(m.thresholdsLastUpdated) > notifierRefreshInterval {
		m.thresholdsLastUpdated = m.clock.Now()
		for _, notifier := range m.thresholdNotifiers {
			if err := notifier.UpdateThreshold(summary); err != nil {
			}
		}
	}

	// 构建资源管理器, 记录各个资源类型 available, capacity, time 三个值.
	observations, statsFunc := makeSignalObservations(summary)

	// 通过thresholdsMet检查observations值和thresholds阈值之间的大小，如果超过阈值则都添加到results；
	thresholds = thresholdsMet(thresholds, observations, false)

	// determine the set of thresholds previously met that have not yet satisfied the associated min-reclaim
	if len(m.thresholdsMet) > 0 {
		thresholdsNotYetResolved := thresholdsMet(m.thresholdsMet, observations, true)
		thresholds = mergeThresholds(thresholds, thresholdsNotYetResolved)
	}

	// 记录时间
	now := m.clock.Now()
	thresholdsFirstObservedAt := thresholdsFirstObservedAt(thresholds, m.thresholdsFirstObservedAt, now)

	// 过滤匹配阈值的条件类型
	nodeConditions := nodeConditions(thresholds)

	// track when a node condition was last observed
	nodeConditionsLastObservedAt := nodeConditionsLastObservedAt(nodeConditions, m.nodeConditionsLastObservedAt, now)

	// 满足 period 的条件类型
	nodeConditions = nodeConditionsObservedSince(nodeConditionsLastObservedAt, m.config.PressureTransitionPeriod, now)

	// 获取满足 period 周期的阈值列表
	thresholds = thresholdsMetGracePeriod(thresholdsFirstObservedAt, now)

	...

	// 再次过滤符合时间的阈值
	thresholds = thresholdsUpdatedStats(thresholds, observations, m.lastObservations)

	// 没有匹配的阈值
	if len(thresholds) == 0 {
		return nil
	}

	// 按照驱逐优先级进行排序
	sort.Sort(byEvictionPriority(thresholds))

	// 获取 thresholds 数组里第一个 thresholdToReclaim, 后面的 rank 方法依赖这次 的 thresholdToReclaim.
	thresholdToReclaim, resourceToReclaim, foundAny := getReclaimableThreshold(thresholds)
	if !foundAny {
		return nil
	}

	// 先尝试进行 imageGC 和 containerGC 垃圾回收清理
	if m.reclaimNodeLevelResources(ctx, thresholdToReclaim.Signal, resourceToReclaim) {
		return nil
	}

	if len(activePods) == 0 {
		return nil
	}

	// 获取 rank 方法
	rank, ok := m.signalToRankFunc[thresholdToReclaim.Signal]
	if !ok {
		return nil
	}

	// 使用 rank 对 pods 进行资源的占比进行排序, 最先被干掉的必然是资源占用最多的.
	rank(activePods, statsFunc)

	// 每次只能删掉一个 pod, 删掉就退出, 等待下次调度循环
	for i := range activePods {
		pod := activePods[i]
		var condition *v1.PodCondition
		...

		// 驱逐 pod
		if m.evictPod(pod, gracePeriodOverride, message, annotations, condition) {
			return []*v1.Pod{pod}
		}
	}
	return nil
}
```

evictPod 函数实现了驱逐 pod, 内部的 killPodFunc 其实是 killPodNow 实现的.

```go
func (m *managerImpl) evictPod(pod *v1.Pod, gracePeriodOverride int64, evictMsg string, annotations map[string]string, condition *v1.PodCondition) bool {
	err := m.killPodFunc(pod, true, &gracePeriodOverride, func(status *v1.PodStatus) {
		status.Phase = v1.PodFailed
		status.Reason = Reason
		status.Message = evictMsg
		if condition != nil {
			podutil.UpdatePodCondition(status, condition)
		}
	})
	return true
}
```

#### 干掉 pod

通过下面的源码可以发现, 干掉 pod 是使用 `podWrokers.UpdatePod` 来实现的, 只需 updateType 指定类型为 `kubetypes.SyncPodKill` 即可.

```go
func killPodNow(podWorkers PodWorkers, recorder record.EventRecorder) eviction.KillPodFunc {
	return func(pod *v1.Pod, isEvicted bool, gracePeriodOverride *int64, statusFn func(*v1.PodStatus)) error {
		timeout := int64(gracePeriod + (gracePeriod / 2))
		minTimeout := int64(10)
		if timeout < minTimeout {
			timeout = minTimeout
		}
		timeoutDuration := time.Duration(timeout) * time.Second

		// open a channel we block against until we get a result
		ch := make(chan struct{}, 1)
		podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			UpdateType: kubetypes.SyncPodKill,
			KillPodOptions: &KillPodOptions{
				CompletedCh:                              ch,
				Evict:                                    isEvicted,
				PodStatusFunc:                            statusFn,
				PodTerminationGracePeriodSecondsOverride: gracePeriodOverride,
			},
		})

		...
	}
}
```

#### 获取节点当前的状态

`summaryProvider.Get()` 方法用来获取当前节点的状态.

```go
func (sp *summaryProviderImpl) Get(ctx context.Context, updateStats bool) (*statsapi.Summary, error) {
	node, err := sp.provider.GetNode()
	if err != nil {
		return nil, fmt.Errorf("failed to get node info: %v", err)
	}
	nodeConfig := sp.provider.GetNodeConfig()
	rootStats, networkStats, err := sp.provider.GetCgroupStats("/", updateStats)
	if err != nil {
		return nil, fmt.Errorf("failed to get root cgroup stats: %v", err)
	}

	// 
	rootFsStats, err := sp.provider.RootFsStats()
	if err != nil {
		return nil, fmt.Errorf("failed to get rootFs stats: %v", err)
	}

	// image 目录的磁盘使用情况
	imageFsStats, err := sp.provider.ImageFsStats(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to get imageFs stats: %v", err)
	}

	var podStats []statsapi.PodStats
	if updateStats {
		podStats, err = sp.provider.ListPodStatsAndUpdateCPUNanoCoreUsage(ctx)
	} 

	// 获取 maxpid 和 进程数等
	rlimit, err := sp.provider.RlimitStats()
	if err != nil {
		return nil, fmt.Errorf("failed to get rlimit stats: %v", err)
	}

	nodeStats := statsapi.NodeStats{
		NodeName:         node.Name,
		CPU:              rootStats.CPU,
		Memory:           rootStats.Memory,
		Network:          networkStats,
		StartTime:        sp.systemBootTime,
		Fs:               rootFsStats,
		Runtime:          &statsapi.RuntimeStats{ImageFs: imageFsStats},
		Rlimit:           rlimit,
		SystemContainers: sp.GetSystemContainersStats(nodeConfig, podStats, updateStats),
	}
	summary := statsapi.Summary{
		Node: nodeStats,
		Pods: podStats,
	}
	return &summary, nil
}
```

#### 有哪些资源被检测

这里的 signal 其实可以理解为资源的描述, 比如 SignalMemoryAvailable 是内存可用的字节.

```go
type Signal string

const (
	// SignalMemoryAvailable is memory available (i.e. capacity - workingSet), in bytes.
	SignalMemoryAvailable Signal = "memory.available"
	// SignalNodeFsAvailable is amount of storage available on filesystem that kubelet uses for volumes, daemon logs, etc.
	SignalNodeFsAvailable Signal = "nodefs.available"
	// SignalNodeFsInodesFree is amount of inodes available on filesystem that kubelet uses for volumes, daemon logs, etc.
	SignalNodeFsInodesFree Signal = "nodefs.inodesFree"
	// SignalImageFsAvailable is amount of storage available on filesystem that container runtime uses for storing images and container writable layers.
	SignalImageFsAvailable Signal = "imagefs.available"
	// SignalImageFsInodesFree is amount of inodes available on filesystem that container runtime uses for storing images and container writable layers.
	SignalImageFsInodesFree Signal = "imagefs.inodesFree"
	// SignalAllocatableMemoryAvailable is amount of memory available for pod allocation (i.e. allocatable - workingSet (of pods), in bytes.
	SignalAllocatableMemoryAvailable Signal = "allocatableMemory.available"
	// SignalPIDAvailable is amount of PID available for pod allocation
	SignalPIDAvailable Signal = "pid.available"
)
```

#### 对pod进行排序的实现

`synchronize` 内部会调用 `buildSignalToRankFunc` 构建 signal 跟 rankFunc 的关系. 

> 再次说民下, 这里的 signal 不是 linux signal 系统信号, 而是监测资源的名称.

`pkg/kubelet/eviction/helpers.go`

```go
// buildSignalToRankFunc returns ranking functions associated with resources
func buildSignalToRankFunc(withImageFs bool) map[evictionapi.Signal]rankFunc {
	signalToRankFunc := map[evictionapi.Signal]rankFunc{
		evictionapi.SignalMemoryAvailable:            rankMemoryPressure,
		evictionapi.SignalAllocatableMemoryAvailable: rankMemoryPressure,
		evictionapi.SignalPIDAvailable:               rankPIDPressure,
	}
	// usage of an imagefs is optional
	if withImageFs {
		// with an imagefs, nodefs pod rank func for eviction only includes logs and local volumes
		signalToRankFunc[evictionapi.SignalNodeFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		signalToRankFunc[evictionapi.SignalNodeFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
		// with an imagefs, imagefs pod rank func for eviction only includes rootfs
		signalToRankFunc[evictionapi.SignalImageFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot}, v1.ResourceEphemeralStorage)
		signalToRankFunc[evictionapi.SignalImageFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot}, resourceInodes)
	} else {
		signalToRankFunc[evictionapi.SignalNodeFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		signalToRankFunc[evictionapi.SignalNodeFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
		signalToRankFunc[evictionapi.SignalImageFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		signalToRankFunc[evictionapi.SignalImageFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
	}
	return signalToRankFunc
}
```

这里实现了一个 multiSorter 结构, 实现了多个方法的排序.

```go
// multiSorter implements the Sort interface, sorting changes within.
type multiSorter struct {
	pods []*v1.Pod
	cmp  []cmpFunc
}

// Sort sorts the argument slice according to the less functions passed to OrderedBy.
func (ms *multiSorter) Sort(pods []*v1.Pod) {
	ms.pods = pods
	sort.Sort(ms)
}

func orderedBy(cmp ...cmpFunc) *multiSorter {
	return &multiSorter{
		cmp: cmp,
	}
}

// Len is part of sort.Interface.
func (ms *multiSorter) Len() int {
	return len(ms.pods)
}

// Swap is part of sort.Interface.
func (ms *multiSorter) Swap(i, j int) {
	ms.pods[i], ms.pods[j] = ms.pods[j], ms.pods[i]
}

// 依次调用注册的 cmp 排序方法
func (ms *multiSorter) Less(i, j int) bool {
	p1, p2 := ms.pods[i], ms.pods[j]
	var k int
	for k = 0; k < len(ms.cmp)-1; k++ {
		cmpResult := ms.cmp[k](p1, p2)
		...
	}
	return ms.cmp[k](p1, p2) < 0
}
```

这里举例说下 rankMemoryPressure 排序的实现, 该排序通过 multiSorter 多排序结构来实现的, 先比较 pod 是否超过的 request 配置, 再比较 pods 的优先级, 最后再对 pods 的 memory usage 使用情况进行排序.

```go
func rankMemoryPressure(pods []*v1.Pod, stats statsFunc) {
	orderedBy(exceedMemoryRequests(stats), priority, memory(stats)).Sort(pods)
}

// 对 pod 的内存使用是否超过请求配置进行比较
func exceedMemoryRequests(stats statsFunc) cmpFunc {
	return func(p1, p2 *v1.Pod) int {
		p1Stats, p1Found := stats(p1)
		p2Stats, p2Found := stats(p2)
		if !p1Found || !p2Found {
			return cmpBool(!p1Found, !p2Found)
		}

		p1Memory := memoryUsage(p1Stats.Memory)
		p2Memory := memoryUsage(p2Stats.Memory)

		// pod1 的内存使用是否超过了 request 配置
		p1ExceedsRequests := p1Memory.Cmp(v1resource.GetResourceRequestQuantity(p1, v1.ResourceMemory)) == 1

		// pod2 的内存使用是否超过了 request 配置
		p2ExceedsRequests := p2Memory.Cmp(v1resource.GetResourceRequestQuantity(p2, v1.ResourceMemory)) == 1

		return cmpBool(p1ExceedsRequests, p2ExceedsRequests)
	}
}

// 对 pod priority 进行比较
func priority(p1, p2 *v1.Pod) int {
	priority1 := corev1helpers.PodPriority(p1)
	priority2 := corev1helpers.PodPriority(p2)
	if priority1 == priority2 {
		return 0
	}
	if priority1 > priority2 {
		return 1
	}
	return -1
}

// 对 pod 内存占用进行比较
func memory(stats statsFunc) cmpFunc {
	return func(p1, p2 *v1.Pod) int {
		p1Stats, p1Found := stats(p1)
		p2Stats, p2Found := stats(p2)
		if !p1Found || !p2Found {
			// prioritize evicting the pod for which no stats were found
			return cmpBool(!p1Found, !p2Found)
		}

		// adjust p1, p2 usage relative to the request (if any)
		p1Memory := memoryUsage(p1Stats.Memory)
		p1Request := v1resource.GetResourceRequestQuantity(p1, v1.ResourceMemory)
		p1Memory.Sub(p1Request)

		p2Memory := memoryUsage(p2Stats.Memory)
		p2Request := v1resource.GetResourceRequestQuantity(p2, v1.ResourceMemory)
		p2Memory.Sub(p2Request)

		return p2Memory.Cmp(*p1Memory)
	}
}
```

下面是 pod 的磁盘占用排序的实现, 直接看源码.

```go
func rankDiskPressureFunc(fsStatsToMeasure []fsStatsType, diskResource v1.ResourceName) rankFunc {
	return func(pods []*v1.Pod, stats statsFunc) {
		orderedBy(exceedDiskRequests(stats, fsStatsToMeasure, diskResource), priority, disk(stats, fsStatsToMeasure, diskResource)).Sort(pods)
	}
}

// 比较 pod 的磁盘占用, 先对比 pod 磁盘的使用情况, 再对比磁盘是否超过 request 的配置
func exceedDiskRequests(stats statsFunc, fsStatsToMeasure []fsStatsType, diskResource v1.ResourceName) cmpFunc {
	return func(p1, p2 *v1.Pod) int {
		p1Stats, p1Found := stats(p1)
		p2Stats, p2Found := stats(p2)
		if !p1Found || !p2Found {
			return cmpBool(!p1Found, !p2Found)
		}

		// 磁盘使用对比
		p1Usage, p1Err := podDiskUsage(p1Stats, p1, fsStatsToMeasure)
		p2Usage, p2Err := podDiskUsage(p2Stats, p2, fsStatsToMeasure)
		if p1Err != nil || p2Err != nil {
			return cmpBool(p1Err != nil, p2Err != nil)
		}

		// 对比磁盘是否超过 request 限制
		p1Disk := p1Usage[diskResource]
		p2Disk := p2Usage[diskResource]
		p1ExceedsRequests := p1Disk.Cmp(v1resource.GetResourceRequestQuantity(p1, diskResource)) == 1
		p2ExceedsRequests := p2Disk.Cmp(v1resource.GetResourceRequestQuantity(p2, diskResource)) == 1
		// prioritize evicting the pod which exceeds its requests
		return cmpBool(p1ExceedsRequests, p2ExceedsRequests)
	}
}
```