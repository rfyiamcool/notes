## 源码分析 kubernetes scheduler framework 插件的实现原理

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

这里选择 `NodeName`, `ImageLocality`, `NodeResourcesFit` 和 `NodeAffinity` 插件来分析的插件实现原理.

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

k8s scheduler 内置插件的 Filter 过滤方法很好理解, 决定节点是否通过预选. 而 Score 打分方法则不好理解, 但通过源码不好反推打分的计算公式是怎么来的.

### NodeAffinity 节点亲和性插件原理

NodeAffinity 是实现节点亲和性的插件, 主要实现了 PreFilter, Filter 和 PreScore, Score 四个方法. PreFilter 和 PreScore 在 NodeAffinity 插件里做 state 传递, 这里重点分析 Filter 和 Score 方法实现.

- `Filter` 用来判断传入的 node 是否匹配 pod 的 `RequiredDuringSchedulingIgnoredDuringExecution` 硬亲和配置, 不适配则返回异常.

- `Score` 用来给 node 打分, 如果节点 labels 适配 pod 的 `preferredDuringSchedulingIgnoredDuringExecution`, 则把 spec.nodeAffinity.weight 累加到 score.

首先看下 pod nodeAffinity 节点亲和性的两个参数.

- `RequiredDuringSchedulingIgnoredDuringExecution` 参数表示调度器只会调度到符合 pod 要求的节点上, 没有符合要求的节点则不进行调度.

- `preferredDuringSchedulingIgnoredDuringExecution` 参数表示调度器会优先尝试寻找满足亲和性规则的节点. 如果实在找不到匹配亲和性的节点, 调度器会选择一个分值高但不满足 pod 亲和性要求的节点.

下面是包含 pod 的亲和性的 pod 配置文件, 可对照该配置来理解 nodeAffinity 插件的 Filter 和 Score 的实现.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-xiaorui-cc
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        preference:
          matchExpressions:
          - key: nodenames
            operator: In
            values:
            - xiaorui.cc
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```

源码位置: `pkg/scheduler/framework/plugins/nodeaffinity/node_affinity.go`

```go
type NodeAffinity struct {
	handle              framework.Handle
	addedNodeSelector   *nodeaffinity.NodeSelector
	addedPrefSchedTerms *nodeaffinity.PreferredSchedulingTerms
}

var _ framework.FilterPlugin = &NodeAffinity{}
var _ framework.ScorePlugin = &NodeAffinity{}
...

// 检查传入的 node 是否匹配该 pod 的 nodeAffinity
func (pl *NodeAffinity) Filter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	node := nodeInfo.Node()
	// node 对象为空则跳出
	if node == nil {
		return framework.NewStatus(framework.Error, "node not found")
	}

	...

	// 获取 prefilter 阶段写入的状态, 其实就是 pod 的 required 配置.
	s, err := getPreFilterState(state)
	if err != nil {
		s = &preFilterState{requiredNodeSelectorAndAffinity: nodeaffinity.GetRequiredNodeAffinity(pod)}
	}

	// 判断 pod 的 required 跟 node 是否适配
	match, _ := s.requiredNodeSelectorAndAffinity.Match(node)
	if !match {
		// 如不适配直接退出
		return framework.NewStatus(framework.UnschedulableAndUnresolvable, ErrReasonPod)
	}

	return nil
}

// 跟 pod 和 node 亲和情况进行打分 score
func (pl *NodeAffinity) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (int64, *framework.Status) {
	...
	node := nodeInfo.Node()

	var count int64
	if pl.addedPrefSchedTerms != nil {
		count += pl.addedPrefSchedTerms.Score(node)
	}

	// 获取 prescore 阶段写入的 state
	s, err := getPreScoreState(state)
	if err != nil {
		// Fallback to calculate preferredNodeAffinity here when PreScore is disabled.
		preferredNodeAffinity, err := getPodPreferredNodeAffinity(pod)
		if err != nil {
			return 0, framework.AsStatus(err)
		}
		s = &preScoreState{
			preferredNodeAffinity: preferredNodeAffinity,
		}
	}

	// 根据 pod 的 preferred 和 node labels 的适配情况, 增加 weight 到 score 分值.
	if s.preferredNodeAffinity != nil {
		count += s.preferredNodeAffinity.Score(node)
	}

	return count, nil
}

func (t *PreferredSchedulingTerms) Score(node *v1.Node) int64 {
	var score int64
	nodeLabels := labels.Set(node.Labels)
	nodeFields := extractNodeFields(node)
	for _, term := range t.terms {
		// 如果 node 匹配 pod preferred, 则增加定义的权重.
		if ok, _ := term.match(nodeLabels, nodeFields); ok {
			score += int64(term.weight)
		}
	}
	return score
}
```