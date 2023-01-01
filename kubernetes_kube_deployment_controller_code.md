# 源码分析 kubernetes deployment controller 的实现原理

## 概述

**DeploymentController**

deployment controller 是 kube-controller-manager 组件中负责 deployment 资源对象的控制器，其通过对 deployment、replicaset、pod 三种资源的监听. 

当三种资源发生变化时会触发 deployment controller 对相应的 deployment 资源进行协调操作，从而完成 deployment 的扩缩容、暂停恢复、更新、回滚、状态 status 更新、所属的旧 replicaset 清理等操作.

**deployment、replicaSet 和 pod 之间的关系**

deployment 的本质是控制 replicaSet, replicaSet 会控制 pod 数, 然后由 controller 驱动各个对象达到期望状态.

## 源码分析

### 实例化 deployment controller

`startDeploymentController` 里实例化 deployment controller 控制器对象, 并且启动控制器.

```go
func startDeploymentController(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {
	dc, err := deployment.NewDeploymentController(
		controllerContext.InformerFactory.Apps().V1().Deployments(),
		controllerContext.InformerFactory.Apps().V1().ReplicaSets(),
		controllerContext.InformerFactory.Core().V1().Pods(),
	)
	go dc.Run(ctx, int(controllerContext.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs))
	return nil, true, nil
}
```

DeploymentController 控制器内通过 informer 监听 deployment, replicaset, pod 三个资源的事件.

```go
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
	dc := &DeploymentController{
		...
	}

	// deployment informer
	dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addDeployment,
		UpdateFunc: dc.updateDeployment,
		DeleteFunc: dc.deleteDeployment,
	})
	// replicaset informer
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addReplicaSet,
		UpdateFunc: dc.updateReplicaSet,
		DeleteFunc: dc.deleteReplicaSet,
	})
	// pod informer
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: dc.deletePod,
	})

	// 核心处理方法
	dc.syncHandler = dc.syncDeployment 
	return dc, nil
}
```

`Run()` 启动控制器, 阻塞等待 informer 的缓存同步完毕, 启动 workers 数量的 worker 协程, 默认为 5 个.

worker 循环的从队列获取数据, 然后交给 syncHandler 处理. 队列里的数据是由 informer 注册的 eventHandler 写入的.

```go
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
	// 等待 informer cache 更新完毕
	if !cache.WaitForNamedCacheSync("deployment", ctx.Done(), dc.dListerSynced, dc.rsListerSynced, dc.podListerSynced) {
		return
	}

	// 默认启动 5 个 worker
	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, dc.worker, time.Second)
	}

	<-ctx.Done()
}

func (dc *DeploymentController) worker(ctx context.Context) {
	// 循环调用
	for dc.processNextWorkItem(ctx) {
	}
}

func (dc *DeploymentController) processNextWorkItem(ctx context.Context) bool {
	// 从队列获取任务, 拿不到任务就阻塞在条件变量上.
	key, quit := dc.queue.Get()
	if quit {
		return false
	}
	defer dc.queue.Done(key)

	// 调用 syncHandler 处理
	err := dc.syncHandler(ctx, key.(string))

	return true
}
```

### syncDeployment 核心处理方法

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212121636716.png)

`syncDeployment` 的代码流程虽然繁杂, 但在每个阶段都有做相关的处理, syncStatusOnly 处理删除操作, sync 处理状态为 pause 暂停的操作, rollback 处理回滚版本的操作, 最后由 rolloutRecreate 和 rolloutRolling 实现升级操作.

其中 syncStatusOnly 和 sync 都是更新 Deployment 的 status.

```go
func (dc *DeploymentController) syncDeployment(ctx context.Context, key string) error {
	// 从 key 中拆解 namespace 和 name 信息
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}

	// 从 informer cache 中获取指定 ns 和 name 的 deployment 对象
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	if errors.IsNotFound(err) {
		return nil
	}

	d := deployment.DeepCopy()

	// 判断 selecor 是否为空
	everything := metav1.LabelSelector{}
	if reflect.DeepEqual(d.Spec.Selector, &everything) {
		if d.Status.ObservedGeneration < d.Generation {
			// 更新 dm 状态
			d.Status.ObservedGeneration = d.Generation
			dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
		}
		return nil
	}

	// 获取 deployment 对应的所有 rs, 通过 LabelSelector 进行匹配
	rsList, err := dc.getReplicaSetsForDeployment(ctx, d)
	if err != nil {
		return err
	}

	// 获取当前 Deployment 对象关联的 pod
	podMap, err := dc.getPodMapForDeployment(d, rsList)
	if err != nil {
		return err
	}

	// 如果该 deployment 处于删除状态，则更新其 status
	if d.DeletionTimestamp != nil {
		return dc.syncStatusOnly(ctx, d, rsList)
	}

	// 检查 pause 状态
	if err = dc.checkPausedConditions(ctx, d); err != nil {
		return err
	}
	// 如果是 pause 状态则进行 sync 同步.
	if d.Spec.Paused {
		return dc.sync(ctx, d, rsList)
	}

	// 检查是否为回滚操作
	if getRollbackTo(d) != nil {
		return dc.rollback(ctx, d, rsList)
	}

	scalingEvent, err := dc.isScalingEvent(ctx, d, rsList)
	if err != nil {
		return err
	}
	// 检查 deployment 是否处于 scale 状态
	if scalingEvent {
		return dc.sync(ctx, d, rsList)
	}

	// 更新操作
	switch d.Spec.Strategy.Type {
	case apps.RecreateDeploymentStrategyType:
		// 重建模式
		return dc.rolloutRecreate(ctx, d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
		// 滚动更新模式
		return dc.rolloutRolling(ctx, d, rsList)
	}
	return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}
```

另外通过 `syncDeployment` 核心函数可以可以看到各个阶段的优先级.

`delete > pause > rollback > scale > rollout`

下面依次讲下 `syncDeployment` 方法内各个阶段具体实现.

#### 删除 deployment

当 DeletionTimestamp 字段存在时, 就意味着需要删除该 deployment, 通过当前 dm 和 rs 对象获取新旧副本集对象.

```go
func (dc *DeploymentController) syncStatusOnly(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
	if err != nil {
		return err
	}

	allRSs := append(oldRSs, newRS)
	return dc.syncDeploymentStatus(ctx, allRSs, newRS, d)
}
```

通过 newRS 和 allRSs 计算 deployment 当前的 status, 然后和 deployment 中的 status 进行比较，若二者有差异则更新 deployment 使用最新的 status.

calculateStatus 计算过程颇为复杂, 简单说就是从新旧 replicaset 里获取最新的 status.

```go
func (dc *DeploymentController) syncDeploymentStatus(ctx context.Context, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, d *apps.Deployment) error {
	newStatus := calculateStatus(allRSs, newRS, d)

	if reflect.DeepEqual(d.Status, newStatus) {
		return nil
	}

	newDeployment := d
	newDeployment.Status = newStatus
	_, err := dc.client.AppsV1().Deployments(newDeployment.Namespace).UpdateStatus(ctx, newDeployment, metav1.UpdateOptions{})
	return err
}
```

值得注意的是,  真正的删除 deployment, rs, pods 操作是放在垃圾回收控制器器 `garbagecollector controller` 完成的. 在其他控制里的删除只是配置 `DeletionTimestamp` 字段，并标记 orphan、background 或者 foreground 删除标签.

### 扩缩容 deployment

当执行 scale 操作时，首先会通过 isScalingEvent 方法判断是否为扩缩容操作，然后通过 dc.sync 方法来执行实际的扩缩容动作。

1. 获取所有的 rs
2. 过滤出 activeRS, rs.Spec.Replicas > 0 的为 activeRS
3. 判断 rs 的 desired 值是否等于 deployment.Spec.Replicas, 若不等于则需要为 rs 进行 scale 操作.

```go
func (dc *DeploymentController) isScalingEvent(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) (bool, error) {
	// 依旧获取新旧所有的 rs 对象.
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
	if err != nil {
		return false, err
	}

	allRSs := append(oldRSs, newRS)
	for _, rs := range controller.FilterActiveReplicaSets(allRSs) {
		// 从 rs annotation 中拿到 deployment.kubernetes.io/desired-replicas 的值
		desired, ok := deploymentutil.GetDesiredReplicasAnnotation(rs)
		if !ok {
			continue
		}
		// 如果不是 replicas 结构, 则需要库容
		if desired != *(d.Spec.Replicas) {
			return true, nil
		}
	}
	return false, nil
}
```

获取新旧两个 replicaset 副本集, newRs 是预期的, oldRss 为当前的存在副本集, 调用 `scale` 扩缩容方法, 最后同步当前的状态.

```go
func (dc *DeploymentController) sync(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)

	// scale 扩缩容
	if err := dc.scale(ctx, d, newRS, oldRSs); err != nil {
		return err
	}
	...

	allRSs := append(oldRSs, newRS)
	// 更新状态
	return dc.syncDeploymentStatus(ctx, allRSs, newRS, d)
}
```

`scale` 方法首先求出需要扩缩容的 pod 数量, 按照策略对 rs 数组进行新旧排序, 为了让每个 rs 都扩缩点, 经过一轮 proportion 计算再对 rs 进行 scale 扩缩容.

```go
func (dc *DeploymentController) scale(ctx context.Context, deployment *apps.Deployment, newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) error {
	...
	// 如果 dm 有配置滚动更新策略, 那么就需要按照策略去对 rs 进行扩缩容
	if deploymentutil.IsRollingUpdate(deployment) {
		allRSs := controller.FilterActiveReplicaSets(append(oldRSs, newRS))
		allRSsReplicas := deploymentutil.GetReplicaCountForReplicaSets(allRSs)

		// 计算最大可以创建出的 pod 数
		allowedSize := int32(0)
		if *(deployment.Spec.Replicas) > 0 {
			// 预期的副本数加上 maxSurge 为最大允许数
			allowedSize = *(deployment.Spec.Replicas) + deploymentutil.MaxSurge(*deployment)
		}

		// 计算需要扩容的 pod 数
		deploymentReplicasToAdd := allowedSize - allRSsReplicas

		var scalingOperation string
		switch {
		case deploymentReplicasToAdd > 0:
			// 若 > 0, 进行排序，把新点的 rs 放到前面, 这样对较新的 rs 扩容更多的 pod
			sort.Sort(controller.ReplicaSetsBySizeNewer(allRSs))
			// up 为扩容动作
			scalingOperation = "up"

		case deploymentReplicasToAdd < 0:
			// 若 <0, 按照新旧进行排序，旧的放前面, 这样可以删除一些较旧的 pod
			sort.Sort(controller.ReplicaSetsBySizeOlder(allRSs))
			// down 为缩容动作
			scalingOperation = "down"
		}

		deploymentReplicasAdded := int32(0)
		nameToSize := make(map[string]int32)
		// 遍历所有的 rs, 计算每个 rs 需要扩容或者缩容到的期望副本数
		for i := range allRSs {
			rs := allRSs[i]

			if deploymentReplicasToAdd != 0 {
				// 估算出 rs 需要扩容或者缩容的副本数
				proportion := deploymentutil.GetProportion(rs, *deployment, deploymentReplicasToAdd, deploymentReplicasAdded)

				// 把计算出来的 proportion 累加到 added
				nameToSize[rs.Name] = *(rs.Spec.Replicas) + proportion
				deploymentReplicasAdded += proportion
			} else {
				nameToSize[rs.Name] = *(rs.Spec.Replicas)
			}
		}

		// 遍历所有的 rs, 第一个最活跃的 rs.Spec.Replicas 加上上面循环中计算出
		// 其他 rs 要加或者减的副本数，然后更新所有 rs 的 rs.Spec.Replicas
		for i := range allRSs {
			rs := allRSs[i]

			// 继续计算累加需要扩缩容的数量
			if i == 0 && deploymentReplicasToAdd != 0 {
				leftover := deploymentReplicasToAdd - deploymentReplicasAdded
				nameToSize[rs.Name] = nameToSize[rs.Name] + leftover
				if nameToSize[rs.Name] < 0 {
					nameToSize[rs.Name] = 0
				}
			}

			// 按照扩容的数量去进行扩缩容
			if _, _, err := dc.scaleReplicaSet(ctx, rs, nameToSize[rs.Name], deployment, scalingOperation); err != nil {
				return err
			}
		}
	}
	return nil
}
```

#### 升级 deployment

k8s 里更新策略有两种, 一种为重建模式, 另一种为滚动更新模式. 当在 yaml 里不定义 `.spec.strategy` 策略时, 默认会给填充 `RollingUpdate`.

```go
switch d.Spec.Strategy.Type {
case apps.RecreateDeploymentStrategyType:
	return dc.rolloutRecreate(ctx, d, rsList, podMap)
case apps.RollingUpdateDeploymentStrategyType:
	return dc.rolloutRolling(ctx, d, rsList)
}
```

##### 滚动升级

`rolloutRolling` 负责负责滚动更新操作, 先尝试对新的 rs 进行扩容, 在扩容后更新状态后就跳出. 等再次 informer 触发事件, 再次进入方法内时, 滚动更新场景下这时当前 pods 数量超过预期值无法 scale up. 后面会尝试对旧的 rs 进行缩容, 缩容完成后需要更新 rollout 状态, 再次跳出.

滚动升级就是这样循环反复地对新 rs 进行扩容, 同时对老的 rs 进行缩容, 一边增一边减, 直到达到预期状态.

```go
func (dc *DeploymentController) rolloutRolling(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	// 依旧获取新旧两个 rs
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, true)
	if err != nil {
		return err
	}
	allRSs := append(oldRSs, newRS)

	// 进行扩容
	scaledUp, err := dc.reconcileNewReplicaSet(ctx, allRSs, newRS, d)
	if err != nil {
		return err
	}
	if scaledUp {
		// 更新状态
		return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
	}

	// 进行缩容
	scaledDown, err := dc.reconcileOldReplicaSets(ctx, allRSs, controller.FilterActiveReplicaSets(oldRSs), newRS, d)
	if err != nil {
		return err
	}
	if scaledDown {
		// 更新状态
		return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
	}

	// 如果滚动完毕, 需要清理
	if deploymentutil.DeploymentComplete(d, &d.Status) {
		if err := dc.cleanupDeployment(ctx, oldRSs, d); err != nil {
			return err
		}
	}

	// 同步状态
	return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
}
```

`reconcileNewReplicaSet` 对 newRs 进行扩容, 其中 `NewRSNewReplicas` 会根据 `RollingUpdate.MaxSurge` 和当前副本数计算得出所需要新增的 pods 数.

在扩容进行完成后, 通过 clientset 来 update 更新 rs 的 annotations 和 spec.replicas 字段, 这里就完事了, rs 的真正维护是依赖 replicaSet controller 实现的.

```go
func (dc *DeploymentController) reconcileNewReplicaSet(ctx context.Context, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
	// 新的 rs 副本数是否达到了预期, 也就是跟 dm 定义的一致, 如果一致则没必要再进行滚动了.
	if *(newRS.Spec.Replicas) == *(deployment.Spec.Replicas) {
		return false, nil
	}

	// 如果 rs 的副本数比预期的多, 那么就需要 scale down 操作, 就是减少副本的操作.
	if *(newRS.Spec.Replicas) > *(deployment.Spec.Replicas) {
		scaled, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, newRS, *(deployment.Spec.Replicas), deployment)
		return scaled, err
	}
	// 计算 newRss 需要的副本数
	newReplicasCount, err := deploymentutil.NewRSNewReplicas(deployment, allRSs, newRS)
	if err != nil {
		return false, err
	}

	// 更新 rs 的 anotation 和 rs.Spec.Replicas 字段
	scaled, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, newRS, newReplicasCount, deployment)
	return scaled, err
}
```

调用 `reconcileOldReplicaSets` 对 oldRS 不断缩容，每次缩容的数量是根据 `RollingUpdate.MaxUnavailable` 和当前可用的pods数量来计算出来的.

```go
func (dc *DeploymentController) reconcileOldReplicaSets(......)   (bool, error) {
	// 计算 oldPodsCount
	oldPodsCount := deploymentutil.GetReplicaCountForReplicaSets(oldRSs)
	if oldPodsCount == 0 {
		return false, nil
	}
    
	// 计算 allPodsCount
	allPodsCount := deploymentutil.GetReplicaCountForReplicaSets(allRSs)
    
	// 计算 maxScaledDown
	maxUnavailable := deploymentutil.MaxUnavailable(*deployment)
	minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
	newRSUnavailablePodCount := *(newRS.Spec.Replicas) - newRS.Status.AvailableReplicas
	maxScaledDown := allPodsCount - minAvailable - newRSUnavailablePodCount
	if maxScaledDown <= 0 {
		return false, nil
	}
    
	// 清理异常的 rs
	oldRSs, cleanupCount, err := dc.cleanupUnhealthyReplicas(oldRSs, deployment, maxScaledDown)
	if err != nil {
		return false, nil
	}
    
	allRSs = append(oldRSs, newRS)
    
	// 缩容 old rs
	scaledDownCount, err := dc.scaleDownOldReplicaSetsForRollingUpdate(allRSs, oldRSs, deployment)
	if err != nil {
		return false, nil
	}
    
	totalScaledDown := cleanupCount + scaledDownCount
	return totalScaledDown > 0, nil
}
```

##### 重新创建

rolloutRecreate 在设计上显得 `简单粗暴`, 正常互联网业务场景下, 应该不会有人用这个模式吧.

`Recreate` 的逻辑不复杂, 首先对旧的 rs 缩容到 0, 等待所有 pods 状态为 not running 后, 再创建新的 rs, 副本数跟 deployment 期望的值一致.

```go
func (dc *DeploymentController) rolloutRecreate(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet, podMap map[types.UID][]*v1.Pod) error {
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
	if err != nil {
		return err
	}
	allRSs := append(oldRSs, newRS)
	activeOldRSs := controller.FilterActiveReplicaSets(oldRSs)

	// 缩容 oldRS
	scaledDown, err := dc.scaleDownOldReplicaSetsForRecreate(ctx, activeOldRSs, d)
	if err != nil {
		return err
	}
	if scaledDown {
		return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
	}
	if oldPodsRunning(newRS, oldRSs, podMap) {
		return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
	}

	// 创建 newRS
	if newRS == nil {
		newRS, oldRSs, err = dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, true)
		if err != nil {
			return err
		}
		allRSs = append(oldRSs, newRS)
	}
	// 扩容 newRS
	if _, err := dc.scaleUpNewReplicaSetForRecreate(ctx, newRS, d); err != nil {
		return err
	}

	// 清理过期的 RS
	if util.DeploymentComplete(d, &d.Status) {
		if err := dc.cleanupDeployment(ctx, oldRSs, d); err != nil {
			return err
		}
	}

	// 同步 deployment 状态
	return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
}
```