# 源码分析 kubernetes scheduler 核心调度器的实现原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301251502086.png)

> 基于 kubernetes `v1.27.0` 源码分析 scheduler 调度器

k8s scheduler 的主要职责是为新创建的 pod 寻找一个最合适的 node 节点, 然后进行 bind node 绑定, 后面 kubelet 才会监听到并创建真正的 pod.

那么问题来了, 如何为 pod 寻找最合适的 node ? 调度器需要经过 predicates 预选和 priority 优选.

- 预选就是从集群的所有节点中根据调度算法筛选出所有可以运行该 pod 的节点集合
- 优选则是按照算法对预选出来的节点进行打分，找到分值最高的节点作为调度节点.

选出最优节点后, 对 apiserver 发起 pod 节点 bind 操作, 其实就是对 pod 的 spec.NodeName 赋值最优节点.

**源码基本调用关系**

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/k8s-scheduler.jpg)

## k8s scheduler 启动入口

k8s scheduler 在启动时会调用注册在 cobra 的 `setup` 来初始化 scheduler 调度器对象.

代码位置: `cmd/kube-scheduler/app/server.go`

```go
func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {
	// 获取默认配置
	if cfg, err := latest.Default(); err != nil {
		return nil, nil, err
	} else {
		opts.ComponentConfig = cfg
	}

	// 验证 scheduler 的配置参数
	if errs := opts.Validate(); len(errs) > 0 {
		return nil, nil, utilerrors.NewAggregate(errs)
	}

	c, err := opts.Config()
	if err != nil {
		return nil, nil, err
	}

	// 配置中填充和调整
	cc := c.Complete()

	...

	// 构建 scheduler 对象
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		cc.DynInformerFactory,
		recorderFactory,
		ctx.Done(),
		...
	)
	if err != nil {
		return nil, nil, err
	}
	...

	return &cc, sched, nil
}
```

实例化 kubernetes scheduler 对象.

代码位置: `pkg/scheduler/scheduler.go`

```go
// New returns a Scheduler
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {


	// 构建 registry 对象, 默认集成了一堆的插件
	registry := frameworkplugins.NewInTreeRegistry()
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}

	podLister := informerFactory.Core().V1().Pods().Lister()
	nodeLister := informerFactory.Core().V1().Nodes().Lister()

	// 实例化快照
	snapshot := internalcache.NewEmptySnapshot()

	// 实例化 queue, 该 queue 为 PriorityQueue.
	podQueue := internalqueue.NewSchedulingQueue(
		profiles[options.profiles[0].SchedulerName].QueueSortFunc(),
		informerFactory,
		...
	)

	// 实例化 cache 缓存
	schedulerCache := internalcache.New(durationToExpireAssumedPod, stopEverything)

	// 实例化 scheduler 对象
	sched := &Scheduler{
		Cache:                    schedulerCache,
		client:                   client,
		nodeInfoSnapshot:         snapshot,
		NextPod:                  internalqueue.MakeNextPodFunc(podQueue),
		StopEverything:           stopEverything,
		SchedulingQueue:          podQueue,
	}
	sched.applyDefaultHandlers()

	// 在 informer 里注册自定义的事件处理方法
	addAllEventHandlers(sched, informerFactory, dynInformerFactory, unionedGVKs(clusterEventMap))

	return sched, nil
}

func (s *Scheduler) applyDefaultHandlers() {
	s.SchedulePod = s.schedulePod
	s.FailureHandler = s.handleSchedulingFailure
}
```

## 注册在 scheduler informer 的回调方法

在 informer 里注册 pod 和 node 资源的回调方法，监听 pod 事件对 queue 和 cache 做回调处理. 监听 node 事件对 cache 做处理.

```go
func addAllEventHandlers(
	sched *Scheduler,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	gvkMap map[framework.GVK]framework.ActionType,
) {
	// 监听 pod 事件，并注册增删改回调方法, 其操作是对 cache 的增删改
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				...
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToCache,
				UpdateFunc: sched.updatePodInCache,
				DeleteFunc: sched.deletePodFromCache,
			},
		},
	)

	// 监听 pod 事件，并注册增删改方法, 对 queue 插入增删改事件.
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				...
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToSchedulingQueue,
				UpdateFunc: sched.updatePodInSchedulingQueue,
				DeleteFunc: sched.deletePodFromSchedulingQueue,
			},
		},
	)

	// 监听 node 事件，注册回调方法，该方法在 cache 里对 node 的增删改查.
	informerFactory.Core().V1().Nodes().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sched.addNodeToCache,
			UpdateFunc: sched.updateNodeInCache,
			DeleteFunc: sched.deleteNodeFromCache,
		},
	)
}
```

## scheudler 启动入口

`Run()` 方法是 k8s scheduler 的启动运行入口, 其内部会循环调用 `scheduleOne` 方法来从优先级队列里获取由 informer 插入的 pod 对象, 调用 `schedulingCycle` 为 pod 选择最优的 node 节点. 如果找到了合适的 node 节点, 则调用 `bindingCycle` 方法来发起 pod 和 node 绑定.

```go
func (sched *Scheduler) Run(ctx context.Context) {
	sched.SchedulingQueue.Run()

	go wait.UntilWithContext(ctx, sched.scheduleOne, 0)

	<-ctx.Done()
	sched.SchedulingQueue.Close()
}

func (sched *Scheduler) scheduleOne(ctx context.Context) {
	// 从 activeQ 中获取需要调度的 pod 数据
	podInfo := sched.NextPod()
	pod := podInfo.Pod

	// 为 pod 选择最优的 node 节点
	scheduleResult, assumedPodInfo, err := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
	if err != nil {
		// 如何没有找到节点，则执行失败方法.
		sched.FailureHandler(schedulingCycleCtx, fwk, assumedPodInfo, err, scheduleResult.reason, scheduleResult.nominatingInfo, start)
		return
	}

	go func() {
		// 像 apiserver 发起 pod -> node 绑定
		status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
		if !status.IsSuccess() {
			sched.handleBindingCycleError(bindingCycleCtx, state, fwk, assumedPodInfo, start, scheduleResult, status)
		}
	}()
}
```

`NextPod` 其实是 `MakeNextPodFunc` 方法. 从 `PriorityQueue` 队列中获取 pod 对象.

```go
func MakeNextPodFunc(queue SchedulingQueue) func() *framework.QueuedPodInfo {
	return func() *framework.QueuedPodInfo {
		podInfo, err := queue.Pop()
		if err == nil {
			return podInfo
		}
		return nil
	}
}

func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
	p.lock.Lock()
	defer p.lock.Unlock()

	for p.activeQ.Len() == 0 {
		// 没有数据则陷入条件变量的等待接口
		p.cond.Wait()
	}
	obj, err := p.activeQ.Pop()
	if err != nil {
		return nil, err
	}
	pInfo := obj.(*framework.QueuedPodInfo)
	pInfo.Attempts++  // 每次 pop 后都增加下 attempts 次数.
	return pInfo, nil
}
```

## schedulingCycle 核心调度周期的实现

`schedulingCycle()` 该方法主要为 pod 选出最优的 node 节点. 先通过预选过程过滤出符合 pod 要求的节点集合, 再通过插件对这些节点进行打分, 使用分值最高的 node 为 pod 调度节点.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301251629071.png)

scheduler 内置各个阶段的各种插件, 预选和优选阶段就是遍历回调插件求出结果.

调度周期 `schedulingCycle` 内关键方法是 `schedulePod`, 其简化流程如下.

1. 先调用 `findNodesThatFitPod` 过滤出符合要求的预选节点.
2. 调用 `prioritizeNodes` 为预选出来的节点进行打分 score.
3. 最后调用 `selectHost` 选择最合适的 node 节点.

```go
func (sched *Scheduler) schedulingCycle(
	...
) (ScheduleResult, *framework.QueuedPodInfo, error) {

	pod := podInfo.Pod

	// 选择节点
	scheduleResult, err := sched.SchedulePod(ctx, fwk, state, pod)
	...

	// 在缓存 cache 中更新状态
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	...
}

func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	// 更新快照
	if err := sched.Cache.UpdateSnapshot(sched.nodeInfoSnapshot); err != nil {
		return result, err
	}

	// 进行预选筛选
	feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
	if err != nil {
		return result, err
	}

	// 预选下来，无节点可以用, 返回错误
	if len(feasibleNodes) == 0 {
		return result, &framework.FitError{
			Pod:         pod,
			NumAllNodes: sched.nodeInfoSnapshot.NumNodes(),
			Diagnosis:   diagnosis,
		}
	}

	// 经过预选就只有一个节点，那么直接返回
	if len(feasibleNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  feasibleNodes[0].Name,
			EvaluatedNodes: 1 + len(diagnosis.NodeToStatusMap),
			FeasibleNodes:  1,
		}, nil
	}

	// 进行优选过程, 对预选出来的节点进行打分
	priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}

	// 从优选中选出最高分的节点
	host, err := selectHost(priorityList)

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(diagnosis.NodeToStatusMap),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}
```

### findNodesThatFitPod

`findNodesThatFitPod` 方法用来实现调度器的预选过程, 其内部调用插件的 PreFilter 和 Filter 方法来筛选出符合 pod 要求的 node 节点集合.

```go
func (sched *Scheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.Diagnosis, error) {
	diagnosis := framework.Diagnosis{
		NodeToStatusMap:      make(framework.NodeToStatusMap),
		UnschedulablePlugins: sets.NewString(),
	}

	// 获取所有的 nodes
	allNodes, err := sched.nodeInfoSnapshot.NodeInfos().List()
	if err != nil {
		return nil, diagnosis, err
	}

	// 调用 framework 的 PreFilter 集合里的插件
	preRes, s := fwk.RunPreFilterPlugins(ctx, state, pod)
	if !s.IsSuccess() {
		// 如果在 prefilter 有异常, 则直接跳出.
		if !s.IsUnschedulable() {
			return nil, diagnosis, s.AsError()
		}
		msg := s.Message()
		diagnosis.PreFilterMsg = msg
		return nil, diagnosis, nil
	}

	...

	// 根据 prefilter 拿到的 node names 获取 node info 对象.
	nodes := allNodes
	if !preRes.AllNodes() {
		nodes = make([]*framework.NodeInfo, 0, len(preRes.NodeNames))
		for n := range preRes.NodeNames {
			nInfo, err := sched.nodeInfoSnapshot.NodeInfos().Get(n)
			if err != nil {
				return nil, diagnosis, err
			}
			nodes = append(nodes, nInfo)
		}
	}

	// 运行 framework 的 filter 插件判断 node 是否可以运行新 pod.
	feasibleNodes, err := sched.findNodesThatPassFilters(ctx, fwk, state, pod, diagnosis, nodes)

	...

	// 调用额外的 extender 调度器来进行预选
	feasibleNodes, err = findNodesThatPassExtenders(sched.Extenders, pod, feasibleNodes, diagnosis.NodeToStatusMap)
	if err != nil {
		return nil, diagnosis, err
	}
	return feasibleNodes, diagnosis, nil
}
```

#### findNodesThatPassFilters 并发执行 Filter 插件方法

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301252324752.png)

`findNodesThatPassFilters` 方法用来遍历执行 framework 里 Filter 插件集合的 Filter 方法.

为了加快执行效率, 减少预选阶段的时延, framework 内部有个 Parallelizer 并发控制器, 启用 16 个协程并发调用插件的 Filter 方法. 在大集群下 nodes 节点会很多, 为了避免遍历全量的 nodes 执行 Filter 和后续的插件逻辑, 这里通过 `numFeasibleNodesToFind` 方法来减少扫描计算的 nodes 数量.

当成功执行 filter 插件方法的数量超过 numNodesToFind 时, 则执行 context cancel(). 这样 framework 并发协程池监听到 ctx 被关闭后, 则不会继续执行后续的任务.

```go
func (sched *Scheduler) findNodesThatPassFilters(
	ctx context.Context,
	fwk framework.Framework,
	pod *v1.Pod,
	nodes []*framework.NodeInfo) ([]*v1.Node, error) {

	numAllNodes := len(nodes)
	// 计算需要扫描的 nodes 数, 避免了超大集群下 nodes 的计算数量.
	// 当集群的节点数小于 100 时, 则直接使用集群的节点数作为扫描数据量
	// 当大于 100 时, 则使用公式计算 `numAllNodes * (50 - numAllNodes/125) / 100`
	numNodesToFind := sched.numFeasibleNodesToFind(fwk.PercentageOfNodesToScore(), int32(numAllNodes))

	feasibleNodes := make([]*v1.Node, numNodesToFind)

	// 如果 framework 未注册 Filter 插件, 则退出.
	if !fwk.HasFilterPlugins() {
		for i := range feasibleNodes {
			feasibleNodes[i] = nodes[(sched.nextStartNodeIndex+i)%numAllNodes].Node()
		}
		return feasibleNodes, nil
	}

	// framework 内置并发控制器, 并发 16 个协程去请求插件的 Filter 方法.
	errCh := parallelize.NewErrorChannel()
	var statusesLock sync.Mutex

	checkNode := func(i int) {
		// 获取 node info 对象
		nodeInfo := nodes[(sched.nextStartNodeIndex+i)%numAllNodes]

		// 遍历执行 framework 的 Filter 插件的 Filter 方法.
		status := fwk.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
		if status.Code() == framework.Error {
			// 如果有错误, 直接把错误传到 errCh 管道里.
			errCh.SendErrorWithCancel(status.AsError(), cancel)
			return
		}
		if status.IsSuccess() {
			// 如果成功执行 Filter 插件的数量超过 numNodesToFind, 则执行 cancel().
			// 当 ctx 被 cancel(), framework 的并发协程池不会继续执行后续的任务.
			length := atomic.AddInt32(&feasibleNodesLen, 1)
			if length > numNodesToFind {
				cancel()
				atomic.AddInt32(&feasibleNodesLen, -1)
			} else {
				feasibleNodes[length-1] = nodeInfo.Node()
			}
		}
		...
	}

	// 并发调用 framework 的 Filter 插件的 Filter 方法.
	fwk.Parallelizer().Until(ctx, numAllNodes, checkNode, frameworkruntime.Filter)
	feasibleNodes = feasibleNodes[:feasibleNodesLen]
	if err := errCh.ReceiveError(); err != nil {
		// 当有错误时, 直接返回.
		statusCode = framework.Error
		return feasibleNodes, err
	}

	// 返回可用的 nodes 列表.
	return feasibleNodes, nil
}
```

framework 的并发库也是通过封装 `workqueue.ParallelizeUntil` 来实现的, 关于 `parallelizer` 的实现原理这里就不做分析了.

`k8s.io/client-go/util/workqueue/parallelizer.go`

#### numFeasibleNodesToFind 计算合理的节点扫描数

`numFeasibleNodesToFind` 方法会根据当前集群的节点数计算出合理的节点扫描数量, 也就是把参与 Filter 的节点范围缩小, 无需全面扫描所有的节点, 这样避免 k8s 集群 nodes 太多时, 造成一些无效的计算资源开销.

`numFeasibleNodesToFind` 策略是这样的, 当集群节点小于 100 时, 则使用集群节点数作为扫描数, 当大于 100 时, 则使用下面的公式计算扫描数. scheudler 的 `percentageOfNodesToScore` 参数默认为 0, 源码中会赋值为 50 %.

```
numAllNodes * (50 - numAllNodes/125) / 100
```

假设当前集群有 `500` 个 nodes 节点, 那么需要执行 Filter 插件方法的 nodee 节点有 `500 * (50 - 500/125) / 100 = 230` 个.

`numFeasibleNodesToFind` 只是表明扫到这个节点数后就结束了, 但如果前面执行插件发生失败时, 自然会加大扫描数.

```go
func (sched *Scheduler) numFeasibleNodesToFind(percentageOfNodesToScore *int32, numAllNodes int32) (numNodes int32) {
	// 当前的集群小于 100 时, 则直接使用集群节点数作为扫描数
	if numAllNodes < minFeasibleNodesToFind {
		return numAllNodes
	}

	// k8s scheduler 的 nodes 百分比默认为 0
	var percentage int32
	if percentageOfNodesToScore != nil {
		percentage = *percentageOfNodesToScore
	} else {
		percentage = sched.percentageOfNodesToScore
	}

	if percentage == 0 {
		percentage = int32(50) - numAllNodes/125
		// 不能小于 5
		if percentage < minFeasibleNodesPercentageToFind {
			percentage = minFeasibleNodesPercentageToFind
		}
	}

	// 需要扫描的节点数
	numNodes = numAllNodes * percentage / 100
	// 如果小于 100, 则使用 100 作为扫描数. 
	if numNodes < minFeasibleNodesToFind {
		return minFeasibleNodesToFind
	}

	return numNodes
}
```

### prioritizeNodes

`prioritizeNodes` 方法为调度器的优选阶段的实现. 其内部会遍历调用 framework 的 PreScore 插件集合里 `PeScore` 方法, 然后再遍历调用 framework 的 Score 插件集合的 `Score` 方法. 经过 Score 打分计算后可以拿到各个 node 的分值.

```go
func prioritizeNodes(
	ctx context.Context,
	extenders []framework.Extender,
	fwk framework.Framework,
	state *framework.CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) ([]framework.NodePluginScores, error) {
	// 如果 extenders 为空和score 插件为空, 则跳出
	if len(extenders) == 0 && !fwk.HasScorePlugins() {
		result := make([]framework.NodePluginScores, 0, len(nodes))
		for i := range nodes {
			result = append(result, framework.NodePluginScores{
				Name:       nodes[i].Name,
				TotalScore: 1,
			})
		}
		return result, nil
	}

	// 在 framework 的 PreScore 插件集合里, 遍历执行插件的 PreSocre 方法
	preScoreStatus := fwk.RunPreScorePlugins(ctx, state, pod, nodes)
	if !preScoreStatus.IsSuccess() {
		// 只有有异常直接退出
		return nil, preScoreStatus.AsError()
	}

	// 在 framework 的 Score 插件集合里, 遍历执行插件的 Socre 方法
	nodesScores, scoreStatus := fwk.RunScorePlugins(ctx, state, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return nil, scoreStatus.AsError()
	}

	klogV := klog.V(10)
	if klogV.Enabled() {
		for _, nodeScore := range nodesScores {
			// 打印插件名字和分值 score
			for _, pluginScore := range nodeScore.Scores {
				klogV.InfoS("Plugin scored node for pod", "pod", klog.KObj(pod), "plugin", pluginScore.Name, "node", nodeScore.Name, "score", pluginScore.Score)
			}
		}
	}

	if len(extenders) != 0 && nodes != nil {
		// 当额外 extenders 调度器不为空时, 则需要计算分值.
		...
		...
	}

	return nodesScores, nil
}
```

### selectHost

`selectHost` 是从优选的 nodes 集合里获取分值 socre 最高的 node. 内部还做了一个小优化, 当相近的两个 node 分值相同时, 则通过随机来选择 node, 目的使 k8s node 的负载更趋于均衡.

```go
func selectHost(nodeScores []framework.NodePluginScores) (string, error) {
	// 如果 nodes 为空, 则返回错误
	if len(nodeScores) == 0 {
		return "", fmt.Errorf("empty priorityList")
	}

	// 直接从头到位遍历 nodeScores 数组, 拿到分值 score 最后的 nodeName.
	maxScore := nodeScores[0].TotalScore
	selected := nodeScores[0].Name
	cntOfMaxScore := 1
	for _, ns := range nodeScores[1:] {
		if ns.TotalScore > maxScore {
			// 当前的分值更大, 则进行赋值.
			maxScore = ns.TotalScore
			selected = ns.Name
			cntOfMaxScore = 1
		} else if ns.TotalScore == maxScore {
			// 当两个 node 的 分值相同时, 
			// 使用随机算法来选择当前和上一个 node.
			cntOfMaxScore++
			if rand.Intn(cntOfMaxScore) == 0 {
				selected = ns.Name
			}
		}
	}

	// 返回分值最高的 node
	return selected, nil
}
```

### PriorityQueue 的实现

`PriorityQueue` 用来实现优先级队列, informer 会条件 pod 到 priorityQueue 队列中, schedulerOne 会从该队列中 pop 对象. 在创建延迟队列时传入一个 `less` 比较方法, 时间最小的 podInfo 放在 heap 的最顶端.

`flushBackoffQCompleted` 会不断的检查 backoff heap 堆顶的元素是否满足条件, 当满足条件把 pod 对象扔到 activeQ 队里, 并激活条件变量. 这里没有采用监听等待堆顶到期时间的方法，而是每隔一秒去检查堆顶的 podInfo 是否已到期 `isPodBackingoff`.

backoff 时长是依赖 podInfo.Attempts 重试次数的，默认情况下重试 5次 是 5s, 最大不能超过 10s. 主调度方法 `scheduleOne` 每次从队列获取 podInfo 时, 它的 Attempts 字段都会加一.

代码位置: `pkg/scheduler/internal/queue/scheduling_queue.go`

```go
const (
	DefaultPodMaxBackoffDuration time.Duration = 10 * time.Second
)

// 实例化优先级队列，该队列中含有 activeQ, podBackoffQ, unschedulablePods集合.
func NewPriorityQueue() {
	pq.podBackoffQ = heap.NewWithRecorder(
		podInfoKeyFunc, 
		pq.podsCompareBackoffCompleted, 
		...
	)
}

// 添加 pod 对象
func (p *PriorityQueue) Add(pod *v1.Pod) error {
	p.lock.Lock()
	defer p.lock.Unlock()

	pInfo := p.newQueuedPodInfo(pod)
	gated := pInfo.Gated

	// 把对象加到 activeQ 队列里.
	if added, err := p.addToActiveQ(pInfo); !added {
		return err
	}
	// 如果该对象在不可调度集合中存在, 则需要在里面删除.
	if p.unschedulablePods.get(pod) != nil {
		p.unschedulablePods.delete(pod, gated)
	}

	// 从 backoffQ 删除 pod 对象
	if err := p.podBackoffQ.Delete(pInfo); err == nil {
		klog.ErrorS(nil, "Error: pod is already in the podBackoff queue", "pod", klog.KObj(pod))
	}

	// 条件变量通知
	p.cond.Broadcast()

	return nil
}

// 获取 pod info 对象
func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
	p.lock.Lock()
	defer p.lock.Unlock()

	// 如果 activeQ 为空, 陷入等待
	for p.activeQ.Len() == 0 {
		if p.closed {
			return nil, fmt.Errorf(queueClosed)
		}
		p.cond.Wait()
	}

	// 从 activeQ 堆顶 pop 对象
	obj, err := p.activeQ.Pop()
	if err != nil {
		return nil, err
	}

	pInfo := obj.(*framework.QueuedPodInfo)
	// 加一
	pInfo.Attempts++
	p.schedulingCycle++
	return pInfo, nil
}

// heap 的比较方法，确保 deadline 最低的在 heap 顶部.
func (p *PriorityQueue) podsCompareBackoffCompleted(podInfo1, podInfo2 interface{}) bool {
	bo1 := p.getBackoffTime(pInfo1)
	bo2 := p.getBackoffTime(pInfo2)
	return bo1.Before(bo2)
}

func (p *PriorityQueue) getBackoffTime(podInfo *framework.QueuedPodInfo) time.Time {
	duration := p.calculateBackoffDuration(podInfo)
	backoffTime := podInfo.Timestamp.Add(duration)
	return backoffTime
}

// backoff duration 随着重试次数不断叠加，但最大不能超过 maxBackoffDuration.
func (p *PriorityQueue) calculateBackoffDuration(podInfo *framework.QueuedPodInfo) time.Duration {
	duration := p.podInitialBackoffDuration // 1s
	for i := 1; i < podInfo.Attempts; i++ {
		if duration > p.podMaxBackoffDuration-duration {
			return p.podMaxBackoffDuration // 10s
		}
		duration += duration
	}
	return duration
}
```

### 如何处理调度失败的 pod

前面有说 kubernetes scheduler 的 `scheduleOne` 作为主循环处理函数，当没有为 pod 找到合适 node 时，会调用 `FailureHandler` 方法.

`FailureHandler()` 是由 `handleSchedulingFailure()` 方法实现. 该逻辑的实现简单说就是把失败的 pod 扔到 podBackoffQ 队列或者 unschedulablePods 集合里.

```go
func (sched *Scheduler) handleSchedulingFailure(ctx context.Context, fwk framework.Framework, podInfo *framework.QueuedPodInfo, err error, reason string, nominatingInfo *framework.NominatingInfo, start time.Time) {
	podLister := fwk.SharedInformerFactory().Core().V1().Pods().Lister()
	cachedPod, e := podLister.Pods(pod.Namespace).Get(pod.Name)
	if e != nil {
	} else {
		if len(cachedPod.Spec.NodeName) != 0 {
			klog.InfoS("Pod has been assigned to node. Abort adding it back to queue.", "pod", klog.KObj(pod), "node", cachedPod.Spec.NodeName)
		} else {
			podInfo.PodInfo, _ = framework.NewPodInfo(cachedPod.DeepCopy())

			// 重新入队列
			if err := sched.SchedulingQueue.AddUnschedulableIfNotPresent(podInfo, sched.SchedulingQueue.SchedulingCycle()); err != nil {
				klog.ErrorS(err, "Error occurred")
			}
		}
	}

	...
}

func (p *PriorityQueue) AddUnschedulableIfNotPresent(pInfo *framework.QueuedPodInfo, podSchedulingCycle int64) error {
	pod := pInfo.Pod

	// 去重判断
	if _, exists, _ := p.activeQ.Get(pInfo); exists {
		return fmt.Errorf("Pod %v is already present in the active queue", klog.KObj(pod))
	}
	// 去重判断
	if _, exists, _ := p.podBackoffQ.Get(pInfo); exists {
		return fmt.Errorf("Pod %v is already present in the backoff queue", klog.KObj(pod))
	}

	// 把没有调度成功的 podInfo 扔到 backoffQ 队列或者 unschedulablePods 集合中.
	if p.moveRequestCycle >= podSchedulingCycle {
		if err := p.podBackoffQ.Add(pInfo); err != nil {
			return fmt.Errorf("error adding pod %v to the backoff queue: %v", klog.KObj(pod), err)
		}
	} else {
		p.unschedulablePods.addOrUpdate(pInfo)
	}

	return nil
}
```

scheduler queue 在启动时会开启两个常驻的协程. 一个协程来管理 `flushBackoffQCompleted()`，每隔一秒来调用一次. 另一个协程来管理 `flushUnschedulablePodsLeftover`, 每隔三十秒来调用一次.

```go
// Run starts the goroutine to pump from podBackoffQ to activeQ
func (p *PriorityQueue) Run() {
	go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
	go wait.Until(p.flushUnschedulablePodsLeftover, 30*time.Second, p.stop)
}
```

`flushBackoffQCompleted` 从 podBackoffQ 获取 podInfo, 然后扔到 activeQ 里，等待 scheduleOne 来调度处理.

```go
func (p *PriorityQueue) flushBackoffQCompleted() {
	activated := false
	for {
		rawPodInfo := p.podBackoffQ.Peek()
		pInfo := rawPodInfo.(*framework.QueuedPodInfo)
		pod := pInfo.Pod
        
		// 从 podBackoffQ 中获取上次调度失败的 podInfo.
		_, err := p.podBackoffQ.Pop()
		if err != nil {
			klog.ErrorS(err, "Unable to pop pod from backoff queue despite backoff completion", "pod", klog.KObj(pod))
			break
		}
        
		// 然后把这 podInfo 再扔到 activeQ 里，让 scheduleOne 主循环来处理.
		if added, _ := p.addToActiveQ(pInfo); added {
			klog.V(5).InfoS("Pod moved to an internal scheduling queue", "pod", klog.KObj(pod), "event", BackoffComplete, "queue", activeQName)
			activated = true
		}
	}

	// 如果有添加成功的，则激活条件变量.
	if activated {
		p.cond.Broadcast()
	}
}
```

`flushUnschedulablePodsLeftover` 加锁遍历 PodInfoMap, 如果某个 pod 的距离上次的调度时间大于60s, 则扔到两个队列中的一个，否则等待下个 30s 再来处理.

```go
func (p *PriorityQueue) flushUnschedulablePodsLeftover() {
	p.lock.Lock()
	defer p.lock.Unlock()

	var podsToMove []*framework.QueuedPodInfo
	for _, pInfo := range p.unschedulablePods.podInfoMap {
		lastScheduleTime := pInfo.Timestamp
		if currentTime.Sub(lastScheduleTime) > p.podMaxInUnschedulablePodsDuration {
			podsToMove = append(podsToMove, pInfo)
		}
	}

	if len(podsToMove) > 0 {
		// 把 podInfo 扔到 activeQ 和 backoffQ 队列中.
		p.movePodsToActiveOrBackoffQueue(podsToMove, UnschedulableTimeout)
	}
}

func (p *PriorityQueue) movePodsToActiveOrBackoffQueue(podInfoList []*framework.QueuedPodInfo, event framework.ClusterEvent) {
	activated := false
	for _, pInfo := range podInfoList {
		pod := pInfo.Pod
		if p.isPodBackingoff(pInfo) {
			// 如果还需要 backoff 退避, 则扔到 podBackoffQ 队列中进行延迟, 后面由 flushBackoffQCompleted 处理.
			if err := p.podBackoffQ.Add(pInfo); err != nil {
			} else {
				p.unschedulablePods.delete(pod, pInfo.Gated)
			}
		} else {
			// 把 podInfo 扔到 activeQ 里, 等待 scheduleOne 调度处理.
			if added, _ := p.addToActiveQ(pInfo); added {
				activated = true
				p.unschedulablePods.delete(pod, gated)
			}
		}
	}
    ...
}
```

## bindingCycle 实现 pod 和 node 绑定

`bindingCycle` 用来实现 pod 和 node 绑定, 其过程为 `prebind -> bind -> postbind`.

```go
func (sched *Scheduler) bindingCycle(
	... {

	assumedPod := assumedPodInfo.Pod

	// 执行插件的 prebind 逻辑
	if status := fwk.RunPreBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost); !status.IsSuccess() {
		return status
	}

	// 执行 bind 插件逻辑
	if status := sched.bind(ctx, fwk, assumedPod, scheduleResult.SuggestedHost, state); !status.IsSuccess() {
		return status
	}

	// 在 bind 绑定后执行收尾操作
	fwk.RunPostBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)

	return nil
}

func (sched *Scheduler) bind(ctx context.Context, fwk framework.Framework, assumed *v1.Pod, targetNode string, state *framework.CycleState) (status *framework.Status) {
	defer func() {
		sched.finishBinding(fwk, assumed, targetNode, status)
	}()

	bound, err := sched.extendersBinding(assumed, targetNode)
	if bound {
		return framework.AsStatus(err)
	}
	return fwk.RunBindPlugins(ctx, state, assumed, targetNode)
}

func (sched *Scheduler) extendersBinding(pod *v1.Pod, node string) (bool, error) {
	for _, extender := range sched.Extenders {
		if !extender.IsBinder() || !extender.IsInterested(pod) {
			continue
		}
		return true, extender.Bind(&v1.Binding{
			ObjectMeta: metav1.ObjectMeta{Namespace: pod.Namespace, Name: pod.Name, UID: pod.UID},
			Target:     v1.ObjectReference{Kind: "Node", Name: node},
		})
	}
	return false, nil
}
```

extender 默认就只有 `DefaultBinder` 插件, 该插件的 bind 逻辑是通过 clientset 对 pod 进行 node 绑定..

代码位置: `pkg/scheduler/framework/plugins/defaultbinder/default_binder.go`

```go
func (b DefaultBinder) Bind(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) *framework.Status {
	binding := &v1.Binding{
		ObjectMeta: metav1.ObjectMeta{Namespace: p.Namespace, Name: p.Name, UID: p.UID},
		Target:     v1.ObjectReference{Kind: "Node", Name: nodeName},
	}
	err := b.handle.ClientSet().CoreV1().Pods(binding.Namespace).Bind(ctx, binding, metav1.CreateOptions{})
	if err != nil {
		return framework.AsStatus(err)
	}
	return nil
}
```

## framework 的实现原理

k8s scheduler 设计了一套 framework 作为调度器的插件系统. 另外在 k8s 中内置了一波插件, 插件是可以在多个扩展点处进行注册. 当调度器在启动时会加载注册插件到 registry, 当 pod 需要创建时, scheduler 通过插件系统过滤出适合的节点集合, 然后在通过实现 socre 打分的插件选出分值最高的节点.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301242208191.png)

### framework 关键挂载点分析

**PreFilter**

在调用 `Filter` 方法前, 执行 `PreFilter` 预筛选逻辑. 插件会检测集群和 pod 是否满足定义的条件. 当不满足时返回错误, 则停止调度周期.

**Filter**

这些插件用于过滤出不能运行该 Pod 的节点.

**PostFilter**

这些插件在 Filter 阶段后调用, 但仅在该 Pod 没有可行的节点时调用.

**PreScore**

插件在执行 `Score` 打分前, 可以先进行 `PreScore` 前置评分工作.

**Score**

对通过预选阶段的节点集合进行打分.

## scheduler 内置插件的实现原理

k8s scheduler 内置了不少插件, 下面是内置的插件.

```go
func getDefaultPlugins() *v1beta3.Plugins {
	plugins := &v1beta3.Plugins{
		MultiPoint: v1beta3.PluginSet{
			Enabled: []v1beta3.Plugin{
				{Name: names.PrioritySort},
				{Name: names.NodeUnschedulable},
				{Name: names.NodeName},
				{Name: names.TaintToleration, Weight: pointer.Int32(3)},
				{Name: names.NodeAffinity, Weight: pointer.Int32(2)},
				{Name: names.NodePorts},
				{Name: names.NodeResourcesFit, Weight: pointer.Int32(1)},
				{Name: names.VolumeRestrictions},
				{Name: names.EBSLimits},
				{Name: names.GCEPDLimits},
				{Name: names.NodeVolumeLimits},
				{Name: names.AzureDiskLimits},
				{Name: names.VolumeBinding},
				{Name: names.VolumeZone},
				{Name: names.PodTopologySpread, Weight: pointer.Int32(2)},
				{Name: names.InterPodAffinity, Weight: pointer.Int32(2)},
				{Name: names.DefaultPreemption},
				{Name: names.NodeResourcesBalancedAllocation, Weight: pointer.Int32(1)},
				{Name: names.ImageLocality, Weight: pointer.Int32(1)},
				{Name: names.DefaultBinder},
			},
		},
	}
	...
	return plugins
}
```

这里选择 `NodeName`, `ImageLocality` 和 `NodeResourcesFit` 插件来分析的插件实现原理.

### NodeName 插件的实现原理

`NodeName` 插件是一个预选插件, 插件实现了 `Filter` 方法, 筛选出跟 pod 相同 nodeName 的 node 节点对象.

源码位置: `pkg/scheduler/framework/plugins/nodename/node_name.go`

```go
type NodeName struct{}

var _ framework.FilterPlugin = &NodeName{}

const (
	Name = names.NodeName

	ErrReason = "node(s) didn't match the requested node name"
)

func (pl *NodeName) Name() string {
	return Name
}

func (pl *NodeName) Filter(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	if nodeInfo.Node() == nil {
		return framework.NewStatus(framework.Error, "node not found")
	}
	if !Fits(pod, nodeInfo) {
		return framework.NewStatus(framework.UnschedulableAndUnresolvable, ErrReason)
	}
	return nil
}

func Fits(pod *v1.Pod, nodeInfo *framework.NodeInfo) bool {
	// 过滤节点, 当返回 true, 该节点通过了初选.
	// 当 pod 没有指定 nodeName 或者 pod 指定的 nodeName 跟传入的 node 名字一致则返回 true.
	return len(pod.Spec.NodeName) == 0 || pod.Spec.NodeName == nodeInfo.Node().Name
}

...
```

### ImageLocality 插件的实现原理

`ImageLocality` 插件实现了 `Score` 分支计算接口, 该插件会计算 pod 关联的容器的镜像在 node 上的状态, 经过各种校验和公式计算后得出一个 score 分值.

至于 `ImageLocality` 插件如何计算 score 分值代码里写清楚了, 但至于为什么要这么计算就看不明白了.

源码位置: `pkg/scheduler/framework/plugins/imagelocality/image_locality.go`

```go
type ImageLocality struct {
	handle framework.Handle
}

var _ framework.ScorePlugin = &ImageLocality{}

const Name = names.ImageLocality

...

func (pl *ImageLocality) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (int64, *framework.Status) {
	// 通过 nodeName 获取 nodeInfo 对象
	nodeInfo, err := pl.handle.SnapshotSharedLister().NodeInfos().Get(nodeName)
	if err != nil {
		return 0, framework.AsStatus(fmt.Errorf("getting node %q from Snapshot: %w", nodeName, err))
	}

	// 获取主机列表
	nodeInfos, err := pl.handle.SnapshotSharedLister().NodeInfos().List()
	if err != nil {
		return 0, framework.AsStatus(err)
	}

	// 当前有多少 node
	totalNumNodes := len(nodeInfos)

	// 经过一堆表达式计算出 node 对应的 score 分支
	score := calculatePriority(sumImageScores(nodeInfo, pod.Spec.Containers, totalNumNodes), len(pod.Spec.Containers))

	return score, nil
}

// 根据 node 当前镜像的状态计算分值.
func calculatePriority(sumScores int64, numContainers int) int64 {
	maxThreshold := maxContainerThreshold * int64(numContainers)
	if sumScores < minThreshold {
		sumScores = minThreshold
	} else if sumScores > maxThreshold {
		sumScores = maxThreshold
	}

	return int64(framework.MaxNodeScore) * (sumScores - minThreshold) / (maxThreshold - minThreshold)
}

// 遍历 pod 里的所有容器, 当容器对应的镜像在 node 中, 则累加计算分值 score.
func sumImageScores(nodeInfo *framework.NodeInfo, containers []v1.Container, totalNumNodes int) int64 {
	var sum int64
	for _, container := range containers {
		// 获取容器的景象在 node 的状态
		if state, ok := nodeInfo.ImageStates[normalizedImageName(container.Image)]; ok {
			sum += scaledImageScore(state, totalNumNodes)
		}
	}
	return sum
}
```

### NodeResourcesFit 插件的实现原理

`NodeResourcesFit` 插件实现了三个 framework 插件接口, 分别是 PreFilterPlugin, FilterPlugin, ScorePlugin 接口.

- `PreFilterPlugin` 只是在 state 缓存里记录 pod 要求的 request 资源.
- `FilterPlugin` 主要用来过滤掉不符合资源要求的 node 节点, 如果当前 node 可申请的 cpu 和 mem 不满足 pod 需求, 则 node 节点会被过滤掉.
- `ScorePlugin` 主要对符合要求通过预选的 node 集合进行打分, 用以得到最优的 node 节点, 其实就是 cpu/mem 资源最充足最空闲的 node 节点.

代码位置: `pkg/scheduler/framework/plugins/noderesources/fit.go`

```go
var _ framework.PreFilterPlugin = &Fit{}
var _ framework.FilterPlugin = &Fit{}
var _ framework.ScorePlugin = &Fit{}

...

// 用来在 cycleState 里记录 pod 所需的 request 资源
func (f *Fit) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) (*framework.PreFilterResult, *framework.Status) {
	cycleState.Write(preFilterStateKey, computePodResourceRequest(pod))
	return nil, nil
}

// 过滤掉不符合 resource request 资源的 node.
func (f *Fit) Filter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	// 获取在 preFilter 阶段写入的 preFilterState
	s, err := getPreFilterState(cycleState)
	if err != nil {
		return framework.AsStatus(err)
	}

	// 判断当前的 node 是否满足 pod 的资源请求需求
	insufficientResources := fitsRequest(s, nodeInfo, f.ignoredResources, f.ignoredResourceGroups)

	// 如果不为空则有异常, 把所有的异常合并在一起, 构建 framework status 对象再返回.
	if len(insufficientResources) != 0 {
		// We will keep all failure reasons.
		failureReasons := make([]string, 0, len(insufficientResources))
		for i := range insufficientResources {
			failureReasons = append(failureReasons, insufficientResources[i].Reason)
		}
		return framework.NewStatus(framework.Unschedulable, failureReasons...)
	}

	// 返回 nil, 说明该节点符合资源要求
	return nil
}

// 判断当前 node 是否满足 pod 的资源要求
func fitsRequest(podRequest *preFilterState, nodeInfo *framework.NodeInfo, ignoredExtendedResources, ignoredResourceGroups sets.String) []InsufficientResource {
	insufficientResources := make([]InsufficientResource, 0, 4)
	allowedPodNumber := nodeInfo.Allocatable.AllowedPodNumber

	// 如果当前 node pods 数超过了最大 pods 数, 则 append.
	if len(nodeInfo.Pods)+1 > allowedPodNumber {
		insufficientResources = append(insufficientResources, InsufficientResource{
			ResourceName: v1.ResourcePods,
			Reason:       "Too many pods",
			Requested:    1,
			Used:         int64(len(nodeInfo.Pods)),
			Capacity:     int64(allowedPodNumber),
		})
	}

	// 如果没有 pod 没有 resource request 配置, 可直接返回
	if podRequest.MilliCPU == 0 &&
		podRequest.Memory == 0 &&
		podRequest.EphemeralStorage == 0 &&
		len(podRequest.ScalarResources) == 0 {
		return insufficientResources
	}

	// 如果当前 node 空闲 cpu 资源不足以运行 pod, 则 append 异常.
	// 当前的 cpu 资源是按照 request 来计算的, 而不是 metrics 实时的.
	if podRequest.MilliCPU > (nodeInfo.Allocatable.MilliCPU - nodeInfo.Requested.MilliCPU) {
		insufficientResources = append(insufficientResources, InsufficientResource{
			ResourceName: v1.ResourceCPU,
			Reason:       "Insufficient cpu",
			Requested:    podRequest.MilliCPU,
			Used:         nodeInfo.Requested.MilliCPU,
			Capacity:     nodeInfo.Allocatable.MilliCPU,
		})
	}
	// 如果当前 node 空闲的内存不满足 pod 的要求, 则 append 异常
	if podRequest.Memory > (nodeInfo.Allocatable.Memory - nodeInfo.Requested.Memory) {
		insufficientResources = append(insufficientResources, InsufficientResource{
			ResourceName: v1.ResourceMemory,
			Reason:       "Insufficient memory",
			Requested:    podRequest.Memory,
			Used:         nodeInfo.Requested.Memory,
			Capacity:     nodeInfo.Allocatable.Memory,
		})
	}
	// 如果当前 node 的存储空间不够 pod 的要求, 则返回错误.
	if podRequest.EphemeralStorage > (nodeInfo.Allocatable.EphemeralStorage - nodeInfo.Requested.EphemeralStorage) {
		insufficientResources = append(insufficientResources, InsufficientResource{
			ResourceName: v1.ResourceEphemeralStorage,
			Reason:       "Insufficient ephemeral-storage",
			Requested:    podRequest.EphemeralStorage,
			Used:         nodeInfo.Requested.EphemeralStorage,
			Capacity:     nodeInfo.Allocatable.EphemeralStorage,
		})
	}

	// 这个是 pod 对扩展资源的要求, 当节点不满足其要求时, append 追加异常.
	for rName, rQuant := range podRequest.ScalarResources {
		...

		if v1helper.IsExtendedResourceName(rName) {
			var rNamePrefix string
			if ignoredResourceGroups.Len() > 0 {
				rNamePrefix = strings.Split(string(rName), "/")[0]
			}
			if ignoredExtendedResources.Has(string(rName)) || ignoredResourceGroups.Has(rNamePrefix) {
				continue
			}
		}

		if rQuant > (nodeInfo.Allocatable.ScalarResources[rName] - nodeInfo.Requested.ScalarResources[rName]) {
			insufficientResources = append(insufficientResources, InsufficientResource{
				ResourceName: rName,
				Reason:       fmt.Sprintf("Insufficient %v", rName),
				Requested:    podRequest.ScalarResources[rName],
				Used:         nodeInfo.Requested.ScalarResources[rName],
				Capacity:     nodeInfo.Allocatable.ScalarResources[rName],
			})
		}
	}

	// 返回所有的资源分配的异常
	return insufficientResources
}
```

`Score` 会根据当前 node 的 cpu/mem/ephemeral-storage 资源空闲情况进行打分, 可分配的资源越多, 那么分值自然就越高.

```go
func (f *Fit) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (int64, *framework.Status) {
	// 获取 nodeInfo 对象
	nodeInfo, err := f.handle.SnapshotSharedLister().NodeInfos().Get(nodeName)
	if err != nil {
		return 0, framework.AsStatus(fmt.Errorf("getting node %q from Snapshot: %w", nodeName, err))
	}

	// 计算分值 score
	return f.score(pod, nodeInfo)
}

func (r *resourceAllocationScorer) score(
	pod *v1.Pod,
	nodeInfo *framework.NodeInfo) (int64, *framework.Status) {

	...
	requested := make([]int64, len(r.resources))
	allocatable := make([]int64, len(r.resources))

	// 遍历 resouces 累加计算 allocatable 和 requested.
	for i := range r.resources {
		// allocatable 是 node 还可以分配的资源
		// req 是 pod 所需要的资源
		alloc, req := r.calculateResourceAllocatableRequest(nodeInfo, pod, v1.ResourceName(r.resources[i].Name))

		// 如果为 0, 通常为扩展资源, 跳过分值计算.
		if alloc == 0 {
			continue
		}
		allocatable[i] = alloc
		requested[i] = req
	}

	// 通过一波公式来计算该 node 分值
	score := r.scorer(requested, allocatable)

	...
	return score, nil
}
```

`NodeResourcesFit` 的 scorer 是具体用来打分的方法, 其内部根据 requested 和 allocatable 两个参数进行打分.

scheduler 对 resource 打分内置三种不同策略, 分别是 LeastAllocated / MostAllocated / RequestedToCapacityRatio.

- LeastAllocated 默认策略, 空闲资源多的分高, 优先调度到空闲资源多的节点上, 各个 node 节点负载均衡.
- MostAllocated 空闲资源少的分高, 优先调度到空闲资源较少的 node 上, 这样 pod 尽量集中起来方便后面资源回收.
- RequestedToCapacityRatio 请求 request 和 node 资源总量的比率低的分高.

```go
const (
	// 默认策略, 空闲资源多的分高, 优先调度到空闲资源多的节点上.
	LeastAllocated ScoringStrategyType = "LeastAllocated"

	// 空闲资源少的分高, 优先调度到空闲资源少的 node 上
	MostAllocated ScoringStrategyType = "MostAllocated"

	// request 和 node 总量的比率低的分高
	RequestedToCapacityRatio ScoringStrategyType = "RequestedToCapacityRatio"
)

// 下面是 NodeResourcesFit 的工厂方法, 根据 strategy 策略使用不同的 scorer 打分策略.
func NewFit(plArgs runtime.Object, h framework.Handle, fts feature.Features) (framework.Plugin, error) {
	// 获取传入参数
	args, ok := plArgs.(*config.NodeResourcesFitArgs)
	...

	strategy := args.ScoringStrategy.Type

	// 根据策略选择 score 插件
	scorePlugin, exists := nodeResourceStrategyTypeMap[strategy]
	if !exists {
		return nil, fmt.Errorf("scoring strategy %s is not supported", strategy)
	}

	return &Fit{
		ignoredResources:         sets.NewString(args.IgnoredResources...),
		ignoredResourceGroups:    sets.NewString(args.IgnoredResourceGroups...),
		handle:                   h,
		resourceAllocationScorer: *scorePlugin(args),
	}, nil
}

// 下面定义了三个 scorer 打分策略.
var nodeResourceStrategyTypeMap = map[config.ScoringStrategyType]scorer{
	config.LeastAllocated: func(args *config.NodeResourcesFitArgs) *resourceAllocationScorer {
		resources := args.ScoringStrategy.Resources
		return &resourceAllocationScorer{
			Name:      string(config.LeastAllocated),
			scorer:    leastResourceScorer(resources),
			resources: resources,
		}
	},
	config.MostAllocated: func(args *config.NodeResourcesFitArgs) *resourceAllocationScorer {
		resources := args.ScoringStrategy.Resources
		return &resourceAllocationScorer{
			Name:      string(config.MostAllocated),
			scorer:    mostResourceScorer(resources),
			resources: resources,
		}
	},
	config.RequestedToCapacityRatio: func(args *config.NodeResourcesFitArgs) *resourceAllocationScorer {
		resources := args.ScoringStrategy.Resources
		return &resourceAllocationScorer{
			Name:      string(config.RequestedToCapacityRatio),
			scorer:    requestedToCapacityRatioScorer(resources, args.ScoringStrategy.RequestedToCapacityRatio.Shape),
			resources: resources,
		}
	},
}
```

k8s scheduler 内置插件的 Filter 过滤方法很好理解, 要么通过, 要么不通过. 而 Score 打分方法则不好理解, 主要单单通过源码不好分析作者为什么要使用那些个计算公式来打分. 结合源码中各个插件的单元测试 ut, 可计算出怎样的参数分分值高.

## 总结

kubernetes scheduler 从 informer 监听新资源变动, 当有新 pod 创建时, scheduler 需要为其分配绑定一个 node 节点. 在调度 Pod 时要经过两个阶段, 即 调度周期 和 绑定周期. 

调度周期分为预选阶段和优选阶段.

- 预选 (Predicates) 是通过插件的 Filter 方法过滤出符合要求的 node 节点
- 优选 (Priorities) 则对预选出来的 node 节点进行打分排序

绑定周期默认只有一个默认插件, 仅给 apiserver 发起 node 跟 pod 绑定的请求.