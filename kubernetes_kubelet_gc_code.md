# 源码分析 kubelet gc 垃圾回收的实现原理

## 参数介绍

kubelet 垃圾回收（Garbage Collection）用来磁盘的空间回收, 它负责自动清理节点上的无用镜像和容器.

kubelet 每隔 1 分钟进行一次容器清理, 删掉挂掉的容器，每隔 5 分钟进行一次镜像清理, 删掉无用的镜像.

**kubelet 容器垃圾回收的参数:**

```
--maximum-dead-containers-per-container: 每个 pod 可以保留几个挂掉的容器, 默认为 1, 也就是每次把挂掉的容器清理掉.
--maximum-dead-containers: 一个节点上最多有多少个挂掉的容器, 默认为 -1, 表示节点不做限制.
--minimum-container-ttl-duration: 容器可被回收的最小生存年龄，默认是 0 分钟，这意味着每个死亡容器都会被立即执行垃圾回收.
```

**kubelet 镜像垃圾回收的参数:**

```
--image-gc-high-threshold: 当磁盘使用率超过 85%, 则进行垃圾回收, 默认为 85%.
--image-gc-low-threshold: 当空间已经小于 80%, 则停止垃圾回收, 默认为 80%.
--minimum-image-ttl-duration: 镜像的最低存留时间, 默认为 2m0s.
```

## 源码解析

### 启动 gc 垃圾回收

启动 kubelet gc 垃圾回收, 每一分钟调用一次容器垃圾回收, 每五分钟进行一次 image 垃圾回收. 当 kubelet `--image-gc-high-threshold` 阈值设为 100 时, 则无需进行 image 垃圾回收.

```go
func (kl *Kubelet) StartGarbageCollection() {
	// 启动 containerGC
	go wait.Until(func() {
		ctx := context.Background()
		if err := kl.containerGC.GarbageCollect(ctx); err != nil {
		}
	}, ContainerGCPeriod, wait.NeverStop)  // 每隔一分钟进行一次容器的垃圾回收清理

	// 如果阈值配置到 100, 那么就不需要 image gc
	if kl.kubeletConfiguration.ImageGCHighThresholdPercent == 100 {
		return
	}

	// 启动 image gc垃圾回收
	go wait.Until(func() {
		ctx := context.Background()
		if err := kl.imageManager.GarbageCollect(ctx); err != nil {
		} else {
			klog.V(vLevel).InfoS("Image garbage collection succeeded")
		}
	}, ImageGCPeriod, wait.NeverStop)  // 每隔五分钟进行一个 image gc 垃圾回收.
}
```

### 容器 gc

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212141947721.png)

`GarbageCollect` 容器垃圾回收的过程:

1. 垃圾爱清理被驱逐的容器
2. 垃圾清理沙箱 sandboxes
3. 清理挂掉 pods 的日志文件

```go
func NewKubeGenericRuntimeManager(
	...
) (KubeGenericRuntime, error) {
	...
	kubeRuntimeManager.containerGC = newContainerGC(runtimeService, podStateProvider, kubeRuntimeManager)
}

func (m *kubeGenericRuntimeManager) GarbageCollect(ctx context.Context, gcPolicy kubecontainer.GCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error {
	return m.containerGC.GarbageCollect(ctx, gcPolicy, allSourcesReady, evictNonDeletedPods)
}

func (cgc *containerGC) GarbageCollect(ctx context.Context, gcPolicy kubecontainer.GCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error {
	errors := []error{}
	// 回收可以被驱逐的容器
	if err := cgc.evictContainers(ctx, gcPolicy, allSourcesReady, evictNonDeletedPods); err != nil {
	}

	// 回收挂掉 pod 的 sandboxes 沙箱
	if err := cgc.evictSandboxes(ctx, evictNonDeletedPods); err != nil {
	}

	// 回收挂掉 pod 的日志目录
	if err := cgc.evictPodLogsDirectories(ctx, allSourcesReady); err != nil {
	}
	return utilerrors.NewAggregate(errors)
}
```

#### evictContainers 

`evictContainers` 清理回收最老一波创建的容器. 获取可以被清理的容器, 条件的状态不是运行中, 且在 minAge 之前就挂掉的容器, minAge 默认是 0s.

如果单个pod容器配置超过 0, 则进行回收多余的容器, 默认为 1, 那么该 pod 内只保留一个容器. 当 `--maximum-dead-containers` 有配置, 且当前 dead 的容器超过该阈值, 则进行回收.

```go
func (cgc *containerGC) evictContainers(ctx context.Context, gcPolicy kubecontainer.GCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error {
	// 获取可以被清理的容器, 状态不是运行中, 且在 minAge 之前就挂掉的容器, minAge 默认是 0s.
	evictUnits, err := cgc.evictableContainers(ctx, gcPolicy.MinAge)
	if err != nil {
		return err
	}

	// allSourceReady 为 true, 则进行回收 
	if allSourcesReady {
		for key, unit := range evictUnits {
			if cgc.podStateProvider.ShouldPodContentBeRemoved(key.uid) || (evictNonDeletedPods && cgc.podStateProvider.ShouldPodRuntimeBeRemoved(key.uid)) {
				cgc.removeOldestN(ctx, unit, len(unit)) // Remove all.
				delete(evictUnits, key)
			}
		}
	}

	// 如果单个pod容器配置超过 0, 则进行回收多余的容器, 默认 kubelet MaxPerPodContainer 为 1, 也就是为该 pod 只保留一个容器.
	if gcPolicy.MaxPerPodContainer >= 0 {
		cgc.enforceMaxContainersPerEvictUnit(ctx, evictUnits, gcPolicy.MaxPerPodContainer)
	}

	// 如果启动时有配置 --maximum-dead-containers 最大 dead 容器数量限制, 且当前挂掉的容器超过了该阈值
	// 则进行回收处理. 但 kubelet 默认配置为 `-1`, 也就是不限制.
	if gcPolicy.MaxContainers >= 0 && evictUnits.NumContainers() > gcPolicy.MaxContainers {
		// 打算按照批次进行回收清理, 最少按 1 个去清理.
		numContainersPerEvictUnit := gcPolicy.MaxContainers / evictUnits.NumEvictUnits()
		if numContainersPerEvictUnit < 1 {
			numContainersPerEvictUnit = 1
		}

		// 每个 pod 里值保存 numContainersPerEvictUnit 数量的容器.
		cgc.enforceMaxContainersPerEvictUnit(ctx, evictUnits, numContainersPerEvictUnit)

		numContainers := evictUnits.NumContainers()
		// 如果当前清理的容器超过 `--maximum-dead-containers` , 则进行清理
		if numContainers > gcPolicy.MaxContainers {
			// 把 evitUnits 嵌套 map + slice 结构转成 slice 结构
			flattened := make([]containerGCInfo, 0, numContainers)
			for key := range evictUnits {
				flattened = append(flattened, evictUnits[key]...)
			}
			// 然后对 slice 进行创建时间排序
			sort.Sort(byCreated(flattened))

			// 从排序数组里, 删除多余个数的容器
			cgc.removeOldestN(ctx, flattened, numContainers-gcPolicy.MaxContainers)
		}
	}
	return nil
}
```

#### evictSandboxes

`evictSandboxes` 清理已经被删除的 pods 的 sandboxes.

```go
func (cgc *containerGC) evictSandboxes(ctx context.Context, evictNonDeletedPods bool) error {
	// 获取当前节点上所有容器
	containers, err := cgc.manager.getKubeletContainers(ctx, true)
	if err != nil {
		return err
	}

	// 获取所有 sandboxes 沙箱
	sandboxes, err := cgc.manager.getKubeletSandboxes(ctx, true)
	if err != nil {
		return err
	}

	// 获取容器的 PodSandboxId, 放到一个 set 集合里.
	sandboxIDs := sets.NewString()
	for _, container := range containers {
		sandboxIDs.Insert(container.PodSandboxId)
	}

	// sandboxesByPod 组织 podUid 和 sendbox 的关系
	sandboxesByPod := make(sandboxesByPodUID)
	for _, sandbox := range sandboxes {
		podUID := types.UID(sandbox.Metadata.Uid)
		sandboxInfo := sandboxGCInfo{
			id:         sandbox.Id,
			createTime: time.Unix(0, sandbox.CreatedAt),
		}

		if sandbox.State == runtimeapi.PodSandboxState_SANDBOX_READY {
			sandboxInfo.active = true
		}

		if sandboxIDs.Has(sandbox.Id) {
			sandboxInfo.active = true
		}

		sandboxesByPod[podUID] = append(sandboxesByPod[podUID], sandboxInfo)
	}

	// 如果 sandboxes 相关的pod都处于 deleted 状态, 删除所有, 否则就保留一个.
	for podUID, sandboxes := range sandboxesByPod {
		if cgc.podStateProvider.ShouldPodContentBeRemoved(podUID) || (evictNonDeletedPods && cgc.podStateProvider.ShouldPodRuntimeBeRemoved(podUID)) {
			cgc.removeOldestNSandboxes(ctx, sandboxes, len(sandboxes))
		} else {
			cgc.removeOldestNSandboxes(ctx, sandboxes, len(sandboxes)-1)
		}
	}
	return nil
}

```

#### evictPodLogsDirectories

`evictPodLogsDirectories` 清理日志空间, 如果某 pod 已经被删除，则可以删除对应的日志空间及软链.

```go
func (cgc *containerGC) evictPodLogsDirectories(ctx context.Context, allSourcesReady bool) error {
	osInterface := cgc.manager.osInterface
	if allSourcesReady {
		// 获取 kubelet 日志里子目录, 默认文件件位置 `/var/log/pods`.
		dirs, err := osInterface.ReadDir(podLogsRootDirectory)
		for _, dir := range dirs {
			name := dir.Name() // 格式为 NAMESPACE_NAME_UID
			podUID := parsePodUIDFromLogsDirectory(name) // 从 name 获取 podUid
			// 如果 poduid 没被删除, 则跳过
			if !cgc.podStateProvider.ShouldPodContentBeRemoved(podUID) {
				continue
			}
			// 删除该 pod 的 logs dir
			osInterface.RemoveAll(filepath.Join(podLogsRootDirectory, name))
		}
	}

	// 回收挂掉 container 的 logs 链接目录
	logSymlinks, _ := osInterface.Glob(filepath.Join(legacyContainerLogsDir, fmt.Sprintf("*.%s", legacyLogSuffix)))
	for _, logSymlink := range logSymlinks {
		if _, err := osInterface.Stat(logSymlink); os.IsNotExist(err) {
			err := osInterface.Remove(logSymlink)
		}
	}
	return nil
}
```

### 镜像 gc

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212141935066.png)

`GarbageCollect` 用来实现 image 的的垃圾回收, 获取磁盘的 stats, 计算磁盘使用率, 如果当磁盘使用率大于 `HighThresholdPercent` 时, 则调用 `freeSpace` 进行空间回收.

```go
func (im *realImageGCManager) GarbageCollect(ctx context.Context) error {
	// 获取磁盘stats
	fsStats, err := im.statsProvider.ImageFsStats(ctx)
	if err != nil {
		return err
	}

	var capacity, available int64
	// 获取总容量
	if fsStats.CapacityBytes != nil {
		capacity = int64(*fsStats.CapacityBytes)
	}
	// 获取可用容量
	if fsStats.AvailableBytes != nil {
		available = int64(*fsStats.AvailableBytes)
	}

	if available > capacity {
		available = capacity
	}

	// 计算磁盘使用率, 求百分比
	usagePercent := 100 - int(available*100/capacity)

	// 若磁盘的使用率大于 `HighThresholdPercent`, 则进行回收镜像.
	if usagePercent >= im.policy.HighThresholdPercent {
		// 计算释放的空间大小
		amountToFree := capacity*int64(100-im.policy.LowThresholdPercent)/100 - available
		freed, err := im.freeSpace(ctx, amountToFree, time.Now())
		if err != nil {
			return err
		}

		if freed < amountToFree {
			return err
		}
	}

	return nil
}
```

`freeSpace` 用来清理没在使用的 docker image 镜像, 已达到释放空间的目的. 先获取正在使用 images 列表, 然后遍历 imageRecords 集合获取未被使用的 images 列表.

对未使用的 images 按照 LRU 进行排序, 最久使用的放在前面，然后遍历 images 列表，删除老镜像，如果释放的空间不够则继续遍历, 直到满足释放空间要求.

```go
func (im *realImageGCManager) freeSpace(ctx context.Context, bytesToFree int64, freeTime time.Time) (int64, error) {
	// 获取正在被运行中的容器使用的 images 列表
	imagesInUse, err := im.detectImages(ctx, freeTime)
	if err != nil {
		return 0, err
	}

	// 获取所有未使用的 images 信息
	images := make([]evictionInfo, 0, len(im.imageRecords))
	for image, record := range im.imageRecords {
		// 如果存在, 说明该镜像正在被使用
		if isImageUsed(image, imagesInUse) {
			continue
		}
		images = append(images, evictionInfo{
			id:          image,
			imageRecord: *record,
		})
	}

	// 对未使用的镜像进行排序, 按照 LRU 算法排列, 最老使用镜像在前面.
	sort.Sort(byLastUsedAndDetected(images))

	spaceFreed := int64(0)
	// 遍历回收没有使用的镜像
	for _, image := range images {
		// 忽略新的镜像
		if image.lastUsed.Equal(freeTime) || image.lastUsed.After(freeTime) {
			continue
		}

		// 忽略不满足最短时间的镜像, 默认为 2m 
		if freeTime.Sub(image.firstDetected) < im.policy.MinAge {
			continue
		}

		// 删除 image 镜像
		err := im.runtime.RemoveImage(ctx, container.ImageSpec{Image: image.id})
		if err != nil {
			continue
		}

		// 在 image 记录缓存中删除对应的 image
		delete(im.imageRecords, image.id)

		// 重新计算需要释放的空间
		spaceFreed += image.size

		// 空间已经释放的差不多了, 则退出循环.
		if spaceFreed >= bytesToFree {
			break
		}
	}

	return spaceFreed, nil
}
```