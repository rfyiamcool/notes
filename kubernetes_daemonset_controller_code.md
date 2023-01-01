## 源码分析 kubernetes daemonset controller 控制器的实现原理

基于 k8s `v1.27.0` 进行源码分析其原理.

### 入口

创建 daemonset controller 控制器对象, 并且在内部使用 informer 注册监听 daemonset, pod, node 资源的变更事件. 注册 eventHandler 没什么可说的，跟其他控制逻辑类型，就是把对象往 queue 里推.

```go
func NewDaemonSetsController(
	...
) (*DaemonSetsController, error) {
	dsc := &DaemonSetsController{
		...
	}

	// daemonset informer
	daemonSetInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dsc.addDaemonset,
		UpdateFunc: dsc.updateDaemonset,
		DeleteFunc: dsc.deleteDaemonset,
	})

	// pod informer
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dsc.addPod,
		UpdateFunc: dsc.updatePod,
		DeleteFunc: dsc.deletePod,
	})

	// node informer
	nodeInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dsc.addNode,
		UpdateFunc: dsc.updateNode,
	})

	return dsc, nil
}
```

`Run()` 启动时要确保各个 informer 同步完毕，`cache.WaitForNamedCacheSync` 的实现很简单，就是周期性的判断所有的 `informer` 是否 synced 同步完成, 一直轮询知道成功同步完毕. 启动多个 `runWorker` 协程, 接着启动一个 gc 回收协程.

`informer` 会把事件生成 key, 怼到队列里, runwokrer 会监听 queue, 然后调用 syncHandler 去完成 daemonset 的同步状态. syncHandler 的具体实现函数是 `syncDaemonSet` 方法.

```go
func (dsc *DaemonSetsController) Run(ctx context.Context, workers int) {
	// 启动时要确保各个 informer 同步完毕， WaitForNamedCacheSync 的实现很简单，就是周期性的判断所有的 informer 是否 synced 同步完成, 一直轮询到成功.
	if !cache.WaitForNamedCacheSync("daemon sets", ctx.Done(), dsc.podStoreSynced, dsc.nodeStoreSynced, dsc.historyStoreSynced, dsc.dsStoreSynced) {
		return
	}

	// 启动多个 runWorker
	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, dsc.runWorker, time.Second)
	}

	// 启动 gc 回收
	go wait.Until(dsc.failedPodsBackoff.GC, BackoffGCInterval, ctx.Done())

	<-ctx.Done()
}

func (dsc *DaemonSetsController) runWorker(ctx context.Context) {
	// 一直循环调用, controller 只有运行，没有退出，当然也不需要退出.
	for dsc.processNextWorkItem(ctx) {
	}
}

func (dsc *DaemonSetsController) processNextWorkItem(ctx context.Context) bool {
	// 从队列里获取 daemonset key, queue 内部有条件变量，拿不到资源会陷入等待.
	dsKey, quit := dsc.queue.Get()
	if quit {
		return false
	}
	defer dsc.queue.Done(dsKey)

	// 执行同步函数, syncHanlder 是 ds 里关键业务处理入口
	err := dsc.syncHandler(ctx, dsKey.(string))
	if err == nil {
		dsc.queue.Forget(dsKey)
		return true
	}

	dsc.queue.AddRateLimited(dsKey)
	return true
}
```

### 同步管理 daemonset 配置

关键函数的调用关系如下:

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212262247518.png)

#### syncDaemonSet

下面是 `syncDaemonSet` 的流程:

1. 从 key 获取 namespace 和 name.
2. 从 ds infomrer lister 里获取 ds 对象.
3. 获取所有的 node 集合.
4. 判断是否满足 expectations 条件, 当不满足预期时, 只更新状态即可, 满足则继续运行.
5. 同步 daemonset 配置.
6. 更新 daemonset status 状态.

```go
func (dsc *DaemonSetsController) syncDaemonSet(ctx context.Context, key string) error {
	// 从 key 拆分 namespace 和 ds name, key 的格式为 namespace/name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}

	// 从 informer list 里获取 ds 对象
	ds, err := dsc.dsLister.DaemonSets(namespace).Get(name)
	if apierrors.IsNotFound(err) { // 找不到则直接跳出
		return nil
	}

	// 获取所有 node 列表
	nodeList, err := dsc.nodeLister.List(labels.Everything())
	if err != nil {
		return err
	}

	// 获取 dskey
	dsKey, err := controller.KeyFunc(ds)


	// 通过 ds 获取 current, old 的 constructHistory
	cur, old, err := dsc.constructHistory(ctx, ds)
	if err != nil {
		return fmt.Errorf("failed to construct revisions of DaemonSet: %v", err)
	}
	hash := cur.Labels[apps.DefaultDaemonSetUniqueLabelKey]

	// 是否满足 expectations 条件, 当不满足预期时, 只更新状态即可.
	if !dsc.expectations.SatisfiedExpectations(dsKey) {
		return dsc.updateDaemonSetStatus(ctx, ds, nodeList, hash, false)
	}

	// 更新 daemonset, 关键函数
	err = dsc.updateDaemonSet(ctx, ds, nodeList, hash, dsKey, old)

	// 更新 daemonset 的 status 状态
	statusErr := dsc.updateDaemonSetStatus(ctx, ds, nodeList, hash, true)
	...
	return nil
}
```

#### updateDaemonSet

分析代码得知 `updateDaemonSet()` 的流程如下:

1. `updateDaemonSet()` 内部通过 `manage` 来同步 daemonset 的状态.
2. 当满足 expectations 且更新策略为 `RollingUpdate` 滚动更新, 则调用 `rollingUpdate` 进行滚动更新.
3. 最后调用 `cleanupHistory` 来进行清理过期的 `ControllerRevision`.

```go
func (dsc *DaemonSetsController) updateDaemonSet(ctx context.Context, ds *apps.DaemonSet, nodeList []*v1.Node, hash, key string, old []*apps.ControllerRevision) error {
	// 处理 daemonset
	err := dsc.manage(ctx, ds, nodeList, hash)
	if err != nil {
		return err
	}

	if dsc.expectations.SatisfiedExpectations(key) {
		switch ds.Spec.UpdateStrategy.Type {
		case apps.OnDeleteDaemonSetStrategyType:
		case apps.RollingUpdateDaemonSetStrategyType:
			// 类型为 rollingUpdate, 则进行滚动更新
			err = dsc.rollingUpdate(ctx, ds, nodeList, hash)
		}
		if err != nil {
			return err
		}
	}

	// 清理不需要的 cleanupHistory
	err = dsc.cleanupHistory(ctx, ds, old)
	if err != nil {
		return fmt.Errorf("failed to clean up revisions of DaemonSet: %w", err)
	}

	return nil
}
```

#### manage

分析代码得知 `manage()` 的流程如下:

1. 通过 `getNodesToDaemonPods()` 获取 daemonset 和 node 关系, 返回的结构为 `map[node][]*v1.Pod`.
2. 遍历 node 集合, 通过 `podsShouldBeOnNode` 获取 node 需要创建和删除的 ds pods 集合. 
3. `getUnscheduledPodsWithoutNode` 获取不能调度的 Pods 集合, 就是遍历 `nodeToDaemonPods`, 把不在 nodeList 集合里的 node 相关的 pods 追加到待删除集合里.
4. 调用 `syncNodes` 来在一些 node 上创建或者删除一些 daemonset pod.

```go
func (dsc *DaemonSetsController) manage(ctx context.Context, ds *apps.DaemonSet, nodeList []*v1.Node, hash string) error {
	// 获取 node 和 ds 的对应关系
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ctx, ds)
	if err != nil {
		return fmt.Errorf("couldn't get node to daemon pod mapping for daemon set %q: %v", ds.Name, err)
	}

	var nodesNeedingDaemonPods, podsToDelete []string
	// 遍历 node 集合
	for _, node := range nodeList {
		// 获取该 node 需要创建或者删除的 pods 集合.
		nodesNeedingDaemonPodsOnNode, podsToDeleteOnNode := dsc.podsShouldBeOnNode(
			node, nodeToDaemonPods, ds, hash)

		// 添加到待新增集合里
		nodesNeedingDaemonPods = append(nodesNeedingDaemonPods, nodesNeedingDaemonPodsOnNode...)

		// 添加到待删除集合里
		podsToDelete = append(podsToDelete, podsToDeleteOnNode...)
	}

	// 获取不能调度的 Pods 集合, 把不在 nodeList 集合里的 node 相关的 pods 追加到待删除集合里.
	podsToDelete = append(podsToDelete, getUnscheduledPodsWithoutNode(nodeList, nodeToDaemonPods)...)

	// 同步操作, 这里会执行添加和删除操作
	if err = dsc.syncNodes(ctx, ds, podsToDelete, nodesNeedingDaemonPods, hash); err != nil {
		return err
	}

	return nil
}
```

#### syncNodes

syncNodes 方法主要是为需要 daemonset 的 node 创建 pod 以及删除多余的 pod, 源码流程如下:

1. 计算本次 createDiff 和 deleteDiff 增减的个数, 每次 sync 最多执行 250 个, 多余的需要等待下次调用解决.
2. 提前先把 createDiff 和 deleteDiff 写入到 expectations 中.
3. 指数级增量批量创建 daemonset pod, 执行完毕后需要依次减少 expectations creation.
4. 直接遍历并发删除 daemonset pod, 执行完毕后需要减少 expectations deletion.

```go
const (
	// BurstReplicas is a rate limiter for booting pods on a lot of pods.
	// 最多单词执行 250 个
	BurstReplicas = 250
)

func (dsc *DaemonSetsController) syncNodes(ctx context.Context, ds *apps.DaemonSet, podsToDelete, nodesNeedingDaemonPods []string, hash string) error {
	// 获取 dsKey
	dsKey, err := controller.KeyFunc(ds)
	if err != nil {
		return fmt.Errorf("couldn't get key for object %#v: %v", ds, err)
	}

	// 需要创建的 ds pod 的个数
	createDiff := len(nodesNeedingDaemonPods)

	// 需要删除删减 ds pod 的个数
	deleteDiff := len(podsToDelete)

	// 一次新增不能超过 250
	if createDiff > dsc.burstReplicas {
		createDiff = dsc.burstReplicas
	}

	// 一次删减不能超过 250
	if deleteDiff > dsc.burstReplicas {
		deleteDiff = dsc.burstReplicas
	}

	// 写入预期值, 声明该 ds key 需要 add 添加 和 del 删除的个数
	dsc.expectations.SetExpectations(dsKey, createDiff, deleteDiff)

	// 收集错误
	errCh := make(chan error, createDiff+deleteDiff)

	generation, err := util.GetTemplateGeneration(ds)
	if err != nil {
		generation = nil
	}

	// 获取 pod 模板
	template := util.CreatePodTemplate(ds.Spec.Template, generation, hash)

	// 求最小, 默认为 1 个
	batchSize := integer.IntMin(createDiff, controller.SlowStartInitialBatchSize)

	// 按照指数级增长进行创建 pod, 依次是 1, 2, 4, 8 ...
	for pos := 0; createDiff > pos; batchSize, pos = integer.IntMin(2*batchSize, createDiff-(pos+batchSize)), pos+batchSize {
		errorCount := len(errCh)
		createWait.Add(batchSize)
		for i := pos; i < pos+batchSize; i++ {
			go func(ix int) {
				// 在 pod templaet 模板里的配置节点的affinity, 也就是指定 node 节点.
				podTemplate := template.DeepCopy()
				podTemplate.Spec.Affinity = util.ReplaceDaemonSetPodNodeNameNodeAffinity(
					podTemplate.Spec.Affinity, nodesNeedingDaemonPods[ix])

				// 创建 pods
				err := dsc.podControl.CreatePods(ctx, ds.Namespace, podTemplate,
					ds, metav1.NewControllerRef(ds, controllerKind))

				if err != nil {
					// 在 expectations 预期里减一 creation.
					dsc.expectations.CreationObserved(dsKey)

					// 传递错误
					errCh <- err
				}
			}(i)
		}
		createWait.Wait()

		skippedPods := createDiff - (batchSize + pos)
		// 创建完了则退出
		if errorCount < len(errCh) && skippedPods > 0 {
			break
		}
	}

	deleteWait := sync.WaitGroup{}
	deleteWait.Add(deleteDiff)
	for i := 0; i < deleteDiff; i++ {
		go func(ix int) {
			// 删除 daemonset pod, 指定 ns 和 podid
			if err := dsc.podControl.DeletePod(ctx, ds.Namespace, podsToDelete[ix], ds); err != nil {
				// 在 expectations 预期里减去一个 deletion 值.
				dsc.expectations.DeletionObserved(dsKey)
			}
		}(i)
	}
	deleteWait.Wait()

	errors := []error{}
	for err := range errCh {
		errors = append(errors, err)
	}
	return utilerrors.NewAggregate(errors)
}
```

##### 创建 pods (CreatePods)

在 pod 里注入引用 `OwnerReferences` 关系, 然后调用 kubeclient 来创建 pod.

```go
func (r RealPodControl) CreatePods(ctx context.Context, namespace string, template *v1.PodTemplateSpec, controllerObject runtime.Object, controllerRef *metav1.OwnerReference) error {
	return r.CreatePodsWithGenerateName(ctx, namespace, template, controllerObject, controllerRef, "")
}

func (r RealPodControl) CreatePodsWithGenerateName(ctx context.Context, namespace string, template *v1.PodTemplateSpec, controllerObject runtime.Object, controllerRef *metav1.OwnerReference, generateName string) error {
	// 注册引用关系
	pod, err := GetPodFromTemplate(template, controllerObject, controllerRef)
	if err != nil {
		return err
	}
	if len(generateName) > 0 {
		pod.ObjectMeta.GenerateName = generateName
	}
	return r.createPods(ctx, namespace, pod, controllerObject)
}

func (r RealPodControl) createPods(ctx context.Context, namespace string, pod *v1.Pod, object runtime.Object) error {
	// 调用 kubeclient 进行 pod 创建
	newPod, err := r.KubeClient.CoreV1().Pods(namespace).Create(ctx, pod, metav1.CreateOptions{})
	if err != nil {
		return err
	}
	accessor, err := meta.Accessor(object)
	...

	return nil
}
```

##### 删除 pod (deletePod)

直接调用 kubeclient 来删除 pod, 按照 namespace 和 podID 来删除.

```go
func (r RealPodControl) DeletePod(ctx context.Context, namespace string, podID string, object runtime.Object) error {
	...

	if err := r.KubeClient.CoreV1().Pods(namespace).Delete(ctx, podID, metav1.DeleteOptions{}); err != nil {
		return fmt.Errorf("unable to delete pods: %v", err)
	}

	....

	return nil
}
```

### rollingUpdate 滚动更新

`rollingUpdate` 的过程相对繁琐一些, 获取 daemonset 的 node 映射关系, 计算 maxSurge, maxUnavailable 值, 获取新旧 pod 对象, 根据条件选择放到新增 nodes 集合还是待删除的 pods 集合. 最后选择使用 `syncNodes` 来同步配置，也就是增删 daemonset pods.

```go
func (dsc *DaemonSetsController) rollingUpdate(ctx context.Context, ds *apps.DaemonSet, nodeList []*v1.Node, hash string) error {
	// 获取 daemonset 的 node 对应关系
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ctx, ds)
	if err != nil {
		return fmt.Errorf("couldn't get node to daemon pod mapping for daemon set %q: %v", ds.Name, err)
	}

	// 计算 maxSurge, maxUnavailable 值
	maxSurge, maxUnavailable, err := dsc.updatedDesiredNodeCounts(ds, nodeList, nodeToDaemonPods)
	if err != nil {
		return fmt.Errorf("couldn't get unavailable numbers: %v", err)
	}

	...

	for nodeName, pods := range nodeToDaemonPods {
		// 获取新旧 pod 对象
		newPod, oldPod, ok := findUpdatedPodsOnNode(ds, pods, hash)
		...
		switch {
		case newPod == nil:
			switch {
			case !podutil.IsPodAvailable(oldPod, ds.Spec.MinReadySeconds, metav1.Time{Time: now}):
				if allowedNewNodes == nil {
					allowedNewNodes = make([]string, 0, len(nodeToDaemonPods))
				}
				// 待添加 nodes 集合
				allowedNewNodes = append(allowedNewNodes, nodeName)
			default:
				if candidateNewNodes == nil {
					candidateNewNodes = make([]string, 0, maxSurge)
				}
				// 待添加 nodes 集合
				candidateNewNodes = append(candidateNewNodes, nodeName)
			}
		default:
			// 待删除 pods 集合
			oldPodsToDelete = append(oldPodsToDelete, oldPod.Name)
		}
	}

	...
	newNodesToCreate := append(allowedNewNodes, candidateNewNodes[:remainingSurge]...)

	// 调用 syncNodes 来创建新 pod 和 清理旧容器.
	return dsc.syncNodes(ctx, ds, oldPodsToDelete, newNodesToCreate, hash)
}
```

### 预期 expectations 的设计

#### expectations 作用

expectations 本质就是个状态结构, 维护一个资源的 add (新增个数) 和 del (删除个数) 两个值, 这个资源在这里是 daemonset, 也可以是 deployment, replicaset, job 等对象. 

expectations 会把资源进行格式化为 `namespace/name` 的字符串, 该字符串作为 expectations 的 key.

#### 那么如何使用 expectations 预期值 ?

在同步 daemonset 配置前, 先判断是否配置了预期值, 通常当不满足预期时, 会进行下一步处理, 否则直接跳出.
操作同步之前, 需要先在 expectations 调用里声明下需要新增和删除的个数. 然后当每次创建完 pod 后, 需要对相应的 add 进行减一. 同理, 删除操作完成后需要对 del 字段进行减一.

```go
type ControllerExpectations struct {
	cache.Store
}

func (r *ControllerExpectations) GetExpectations(controllerKey string) (*ControlleeExpectations, bool, error) {
	exp, exists, err := r.GetByKey(controllerKey)
	if err == nil && exists {
		return exp.(*ControlleeExpectations), true, nil
	}
	return nil, false, err
}

func (r *ControllerExpectations) SatisfiedExpectations(controllerKey string) bool {
	if exp, exists, err := r.GetExpectations(controllerKey); exists {
		if exp.Fulfilled() {
			return true
		} else if exp.isExpired() {
			return true
		} else {
			return false
		}
	}
	...
}

func (r *ControllerExpectations) CreationObserved(controllerKey string) {
	r.LowerExpectations(controllerKey, 1, 0)
}

func (r *ControllerExpectations) DeletionObserved(controllerKey string) {
	r.LowerExpectations(controllerKey, 0, 1)
}

func (r *ControllerExpectations) LowerExpectations(controllerKey string, add, del int) {
	if exp, exists, err := r.GetExpectations(controllerKey); err == nil && exists {
		exp.Add(int64(-add), int64(-del))
	}
}
```

`ControlleeExpectations` 结构体维护了 add 和 del 数字, ds controller 调用 `add()` 方法对预期值进行递减. 当 add 和 del 等于小于 0 时, 则可以认定完事了.

```go
type ControlleeExpectations struct {
	add       int64
	del       int64
	key       string
	timestamp time.Time
}

func (e *ControlleeExpectations) Add(add, del int64) {
	atomic.AddInt64(&e.add, add)
	atomic.AddInt64(&e.del, del)
}

func (e *ControlleeExpectations) Fulfilled() bool {
	return atomic.LoadInt64(&e.add) <= 0 && atomic.LoadInt64(&e.del) <= 0
}

func (e *ControlleeExpectations) GetExpectations() (int64, int64) {
	return atomic.LoadInt64(&e.add), atomic.LoadInt64(&e.del)
}

func (exp *ControlleeExpectations) isExpired() bool {
	return clock.RealClock{}.Since(exp.timestamp) > ExpectationsTimeout
}
```

### 更新 daemonset 状态

```go
func (dsc *DaemonSetsController) updateDaemonSetStatus(ctx context.Context, ds *apps.DaemonSet, nodeList []*v1.Node, hash string, updateObservedGen bool) error {
	// 获取 daemonset pod 跟 node 映射关系
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ctx, ds)
	if err != nil {
		return fmt.Errorf("couldn't get node to daemon pod mapping for daemon set %q: %v", ds.Name, err)
	}

	var desiredNumberScheduled, currentNumberScheduled, numberMisscheduled, numberReady, updatedNumberScheduled, numberAvailable int

	// 遍历 node 集合
	for _, node := range nodeList {
		shouldRun, _ := NodeShouldRunDaemonPod(node, ds)
		scheduled := len(nodeToDaemonPods[node.Name]) > 0

		if shouldRun {
			desiredNumberScheduled++
			if !scheduled {
				continue
			}

			currentNumberScheduled++
			daemonPods, _ := nodeToDaemonPods[node.Name]
			sort.Sort(podByCreationTimestampAndPhase(daemonPods))
			pod := daemonPods[0]
			if podutil.IsPodReady(pod) {
				numberReady++
				if podutil.IsPodAvailable(pod, ds.Spec.MinReadySeconds, metav1.Time{Time: now}) {
					numberAvailable++
				}
			}

			if util.IsPodUpdated(pod, hash, generation) {
				updatedNumberScheduled++
			}
		} else {
			if scheduled {
				numberMisscheduled++
			}
		}
	}
	numberUnavailable := desiredNumberScheduled - numberAvailable

	// 更新 daemonset 状态
	err = storeDaemonSetStatus(ctx, dsc.kubeClient.AppsV1().DaemonSets(ds.Namespace), ds, desiredNumberScheduled, currentNumberScheduled, numberMisscheduled, numberReady, updatedNumberScheduled, numberAvailable, numberUnavailable, updateObservedGen)
	if err != nil {
		return fmt.Errorf("error storing status for daemon set %#v: %w", ds, err)
	}

	// 当就绪计数不相等时, 继续延迟入队
	if ds.Spec.MinReadySeconds > 0 && numberReady != numberAvailable {
		dsc.enqueueDaemonSetAfter(ds, time.Duration(ds.Spec.MinReadySeconds)*time.Second)
	}
	return nil
}
```