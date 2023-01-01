# 源码分析 kubernetes kubelet pod 管理的实现原理

## 源码分析

### 启动入口

实例化 kubelet 服务实例, 并启动 syncLoop 调度核心.

代码位置: `pkg/kubelet/kubelet.go`

```go
func RunKubelet(kubeServer *options.KubeletServer, kubeDeps *kubelet.Dependencies, runOnce bool) error {
	// 创建且实例化 kubelet 服务
	k, err := createAndInitKubelet(kubeServer,
		kubeDeps,
		hostname,
		hostnameOverridden,
		nodeName,
		nodeIPs)
	if err != nil {
		return fmt.Errorf("failed to create kubelet: %w", err)
	}
	podCfg := kubeDeps.PodConfig

	// 启动 kubelet
	startKubelet(k, podCfg, &kubeServer.KubeletConfiguration, kubeDeps, kubeServer.EnableServer)
}

// 启动 kubelet
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, ...) {
	go k.Run(podCfg.Updates())
}

func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	ctx := context.Background()

	...

	kl.pleg.Start()

	// kubelet 的核心调度代码
	kl.syncLoop(ctx, updates, kl)
}
```

updates 这个管道很重要，在初始化 `kubelet` 对象时, 内部通过 `makePodSourceConfig` 方法可以监听 apiserver 的配置更新, 把更新的事件扔到这个 updates 管道里. 另外它还监听了文件及http的接口. 

调用关系: `createAndInitKubelet -> NewMainKubelet -> makePodSourceConfig`

```go
func makePodSourceConfig(kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *Dependencies, nodeName types.NodeName, nodeHasSynced func() bool) (*config.PodConfig, error) {
	// source of all configuration
	cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kubeDeps.Recorder, kubeDeps.PodStartupLatencyTracker)

	// define file config source
	if kubeCfg.StaticPodPath != "" {
		klog.InfoS("Adding static pod path", "path", kubeCfg.StaticPodPath)
		config.NewSourceFile(kubeCfg.StaticPodPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(ctx, kubetypes.FileSource))
	}

	// define url config source
	if kubeCfg.StaticPodURL != "" {
		klog.InfoS("Adding pod URL with HTTP header", "URL", kubeCfg.StaticPodURL, "header", manifestURLHeader)
		config.NewSourceURL(kubeCfg.StaticPodURL, manifestURLHeader, nodeName, kubeCfg.HTTPCheckFrequency.Duration, cfg.Channel(ctx, kubetypes.HTTPSource))
	}

	if kubeDeps.KubeClient != nil {
		klog.InfoS("Adding apiserver pod source")
		config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, nodeHasSynced, cfg.Channel(ctx, kubetypes.ApiserverSource))
	}
	return cfg, nil
}
```

### 调度核心 (syncLoop)

syncLoop 是 kubelet 的调度核心, 内部定义了两个定时器, 一个用来同步的 syncTicker 定时器, 一个是 用来清理异常 pods 的 housekeepingTicker 定时器.

循环调度 syncLoopIteration 方法.

```go
func (kl *Kubelet) syncLoop(ctx context.Context, updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	klog.InfoS("Starting kubelet main sync loop")

	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()

	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()

	plegCh := kl.pleg.Watch()

	...
	for {
		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(ctx, updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```

kubelet 的 pods 同步逻辑都在 `syncLoopIteration` 这里. `syncLoopIteration` 同时监听下面的 chan, 根据事件做不同的处理.

- configCh: 监听 file, http, apiserver 的事件更新
- syncCh: 定时器管道, 每隔一秒去同步最新保存的 pod 状态
- houseKeepingCh: housekeeping 事件的管道，做 pod 清理工作
- plegCh: 该信息源由 kubelet 对象中的 pleg 子模块提供，该模块主要用于周期性地向 container runtime 查询当前所有容器的状态.
- livenessManager.Updates: 健康检查发现某个 pod 不可用, kubelet 将根据 Pod 的 restartPolicy 自动执行正确的操作

```go
func (kl *Kubelet) syncLoopIteration(ctx context.Context, configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {

	select {
	case u, open := <-configCh: // 来自 apiserver 的 pod 事件
		if !open {
			return false
		}

		switch u.Op {
		case kubetypes.ADD: // 添加
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE: // 更新
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.RECONCILE: // 协调
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE: // 删除
			handler.HandlePodUpdates(u.Pods)
		default:
			klog.ErrorS(nil, "Invalid operation type received", "operation", u.Op)
		}

		kl.sourcesReady.AddSource(u.Source)

	case e := <-plegCh: // 由 pleg 子模块上报的事件, pleg 会扫描当前所有容器, 当状态发生变更时发出事件
		if isSyncPodWorthy(e) {
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				handler.HandlePodSyncs([]*v1.Pod{pod})
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	case <-syncCh: // 由定时器触发更新
		podsToSync := kl.getPodsToSync()
		handler.HandlePodSyncs(podsToSync)
	case update := <-kl.livenessManager.Updates(): // 当 liveness 状态发生变更时
		if update.Result == proberesults.Failure {
			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
		}
	case update := <-kl.readinessManager.Updates(): // 当 readiness 状态变更时
		kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)

		handleProbeSync(kl, update, handler, "readiness", status)
	case update := <-kl.startupManager.Updates(): // 当 startup 状态变更时
		kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)
		handleProbeSync(kl, update, handler, "startup", status)

	case <-housekeepingCh: // 定时器触发
		handler.HandlePodCleanups(ctx)
	}
	return true
}
```

### 增加 pod 流程 (HandlePodAdditions)

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212181730298.png)

`HandlePodAdditions` 是创建 Pod 的核心代码. 首先对传入的 pods 进行排序, 保证先提交创建请求的 pod 被先创建, 最后调用 dispatchWork 来创建 pod.

另外静态 pod 是走 `handleMirrorPod` 流程.

```go
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	start := kl.clock.Now()

	// 按照创建事件的先后对传入的 pods 进行排序, 保证是 fifo 的模型. 
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	for _, pod := range pods {
		existingPods := kl.podManager.GetPods()

		// 把 pod 添加到 podManager 里
		kl.podManager.AddPod(pod)

		// 判断是否是静态 pod
		if kubetypes.IsMirrorPod(pod) {
			kl.handleMirrorPod(pod, start)
			continue
		}

		// 在 dispatchWork 里去做 pod 操作, 这里操作为创建 pod
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
	}
}
```

另外, 主调度核心里对 pod 进行增删改操作, 其实最后都会跳到 dispatchWork 方法上.

该方法里主要定义了类型 `kubetypes.SyncPodType`, 然后调用 `podWrokers.UpdatePod` 异步操作 pod.

代码位置: `pkg/kubelet/pod_workers.go`

```go
func (kl *Kubelet) dispatchWork(pod *v1.Pod, syncType kubetypes.SyncPodType, mirrorPod *v1.Pod, start time.Time) {
	// Run the sync in an async worker.
	kl.podWorkers.UpdatePod(UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		StartTime:  start,
	})
}
```

由于 kubelet 创建 pod 容器路径太深, 索性忽略下面的路径，直接跳到 syncPod 方法中.

`podWorkers.UpdatePod -> podWorkers.managePodLoop -> podWorkers.syncPodFn -> kubelet.syncPod`

#### 为 pod 做准备工作及创建 pod ( syncPod )

kubelet.syncPod 主要用来实现 pod 资源的创建, 内部会做好 pod 的准备工作, 流程如下:

1. 更新 pod 状态到 statusManager
2. 检查网络插件是否就绪
3. 把 pod 注册到 secretManager 和 configMapManager 管理器里
4. 创建更新 cgroups 策略
5. 为静态pod创建一个 mirror pod
6. 为 pod 实例化数据目录
7. 配置 volume 挂载
8. 为 pod 拉取 secrets 配置
9. 为 pod 添加探针检测
10. 调用容器的运行时 SyncPod 完成容器重建

代码位置: `pkg/kubelet/kubelet.go : syncPod()`

```go
func (kl *Kubelet) syncPod(_ context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) {
	// 更新 pod 状态到 statusManager
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// 检查网络插件是否就绪
	if err := kl.runtimeState.networkErrors(); err != nil && !kubecontainer.IsHostNetworkPod(pod) {
		return false, fmt.Errorf("%s: %v", NetworkNotReadyErrorMsg, err)
	}

	// 把 pod 注册到 secretManager 和 configMapManager 管理器里
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		if kl.secretManager != nil {
			kl.secretManager.RegisterPod(pod)
		}
		if kl.configMapManager != nil {
			kl.configMapManager.RegisterPod(pod)
		}
	}

	pcm := kl.containerManager.NewPodContainerManager()
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		// 创建更新 cgroups 策略
		if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
			if !pcm.Exists(pod) {
				if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
				}
			}
		}
	}

	// 为静态 pod 创建一个 mirror pod, 
	if kubetypes.IsStaticPod(pod) {
		deleted := false
		// 如果不为空, 则需要清理
		if mirrorPod != nil {
			if mirrorPod.DeletionTimestamp != nil || !kl.podManager.IsMirrorPodOf(mirrorPod, pod) {
				podFullName := kubecontainer.GetPodFullName(pod)
				deleted, err = kl.podManager.DeleteMirrorPod(podFullName, &mirrorPod.ObjectMeta.UID)
			}
		}
		// 如果为空, 则需要为静态pod创建 mirror pod
		if mirrorPod == nil || deleted {
			kl.podManager.CreateMirrorPod(pod)
		}
	}

	// 为 pod 实例化数据目录
	if err := kl.makePodDataDirs(pod); err != nil {
		return false, err
	}

	// 配置 volume 挂载
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		// 同步等待 volumes 完成
		if err := kl.volumeManager.WaitForAttachAndMount(pod); err != nil {
			return false, err
		}
	}

	// 为 pod 拉取 secrets 配置
	pullSecrets := kl.getPullSecretsForPod(pod)

	// 为 pod 添加探针检测
	kl.probeManager.AddPod(pod)

	// 调用容器的运行时 SyncPod 完成容器重建
	result := kl.containerRuntime.SyncPod(ctx, pod, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	if err := result.Error(); err != nil {
		return false, nil
	}

	return false, nil
}
```

#### 创建启动 pod 内的容器 (SyncPod)

`SyncPod` 会依次创建启动 pod 内的容器, 流程如下:

1. 清理容器
2. 创建 sandbox 容器, sandbox 其实就是 pause 容器.
3. 创建临时容器 
4. 创建 init 容器
5. 创建业务容器
6. 完成容器创建

代码位置: `pkg/kubelet/kuberuntime/kuberuntime_manager.go`

```go
func (m *kubeGenericRuntimeManager) SyncPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	// 计算 sandbox 和 容器发生的变动
	podContainerChanges := m.computePodActions(pod, podStatus)
	// 如果 sandbox 发生变动, 则干
	if podContainerChanges.KillPod {
		// 干掉 pod
		killResult := m.killPodWithSyncResult(ctx, pod, kubecontainer.ConvertPodStatusToRunningPod(m.runtimeName, podStatus), nil)
		result.AddPodSyncResult(killResult)

		if podContainerChanges.CreateSandbox {
			// 并清理 init 容器
			m.purgeInitContainers(ctx, pod, podStatus)
		}
	} else {
		// 干掉不能运行的容器
		for containerID, containerInfo := range podContainerChanges.ContainersToKill {
			if err := m.killContainer(ctx, pod, containerID, containerInfo.name, containerInfo.message, containerInfo.reason, nil); err != nil {
				return
			}
		}
	}

	var podIPs []string
	if podStatus != nil {
		podIPs = podStatus.IPs
	}

	// 为 pod 创建 sandbox
	podSandboxID := podContainerChanges.SandboxID
	if podContainerChanges.CreateSandbox {
		// 创建 pod sandbox
		podSandboxID, msg, err = m.createPodSandbox(ctx, pod, podContainerChanges.Attempt)
		if err != nil {
			...
			return
		}

		// 如果 pod 网络是 host 模式，容器也相同；其他情况下，容器会使用 None 网络模式，让 kubelet 的网络插件自己进行网络配置
		if !kubecontainer.IsHostNetworkPod(pod) {
			podIPs = m.determinePodSandboxIPs(pod.Namespace, pod.Name, resp.GetStatus())
			klog.V(4).InfoS("Determined the ip for pod after sandbox changed", "IPs", podIPs, "pod", klog.KObj(pod))
		}
	}

	podIP := ""
	if len(podIPs) != 0 {
		podIP = podIPs[0]
	}

	// 获取 pod sandbox 的配置, 里面有 metadata,clusterDNS,容器的端口映射等.
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, podContainerChanges.Attempt)
	if err != nil {
		configPodSandboxResult.Fail(kubecontainer.ErrConfigPodSandbox, message)
		return
	}

	// 定义一个匿名的启动各种类型容器的方法
	start := func(ctx context.Context, typeName, metricLabel string, spec *startSpec) error {
		...

		// 启动容器
		if msg, err := m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs); err != nil {
			...
			return err
		}

		return nil
	}

	// 启动临时容器, 临时容器不会自动重启, 通常配合 kubectl debug 调试使用.
	for _, idx := range podContainerChanges.EphemeralContainersToStart {
		start(ctx, "ephemeral container", metrics.EphemeralContainer, ephemeralContainerStartSpec(&pod.Spec.EphemeralContainers[idx]))
	}

	// 启动 init 容器
	if container := podContainerChanges.NextInitContainerToStart; container != nil {
		// Start the next init container.
		if err := start(ctx, "init container", metrics.InitContainer, containerStartSpec(container)); err != nil {
			return
		}

		// Successfully started the container; clear the entry in the failure
		klog.V(4).InfoS("Completed init container for pod", "containerName", container.Name, "pod", klog.KObj(pod))
	}

	// 启动业务容器
	for _, idx := range podContainerChanges.ContainersToStart {
		// 调用上方定义的 start 匿名函数.
		start(ctx, "container", metrics.Container, containerStartSpec(&pod.Spec.Containers[idx]))
	}

	return
}
```

#### 真正去创建启动容器 (startContainer)

`startContainer` 是真正创建容器的方法, 流程如下:

1. 拉取容器镜像, 不存在则直接拉取.
2. 配置新容器的重启次数为 0, 重启被干掉后重建的容器, 重启次数会累加.
3. 为新容器实例化日志目录
3. 配置容器的日志目录
4. 生成容器配置
5. 创建容器
6. 启动容器
7. 日志目录做软连接
8. 执行 post start hook, 出错则需要干掉启动的容器.

代码位置: `pkg/kubelet/kuberuntime/kuberuntime_container.go`

```go
func (m *kubeGenericRuntimeManager) startContainer(ctx context.Context, podSandboxID string, podSandboxConfig *runtimeapi.PodSandboxConfig, spec *startSpec, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, podIP string, podIPs []string) (string, error) {
	container := spec.container

	// 拉取容器镜像, 不存在拉取
	imageRef, msg, err := m.imagePuller.EnsureImageExists(ctx, pod, container, pullSecrets, podSandboxConfig)
	if err != nil {
		return msg, err
	}

	// 配置新容器的重启次数为 0, 重启被干掉后重建的容器, 重启次数会累加.
	restartCount := 0
	containerStatus := podStatus.FindContainerStatusByName(container.Name)
	if containerStatus != nil {
		restartCount = containerStatus.RestartCount + 1
	} else {
		// 新容器则需要创建容器的日志目录
		logDir := BuildContainerLogsDirectory(pod.Namespace, pod.Name, pod.UID, container.Name)
	}

	// 根据传递进来的参数生成容器配置
	containerConfig, cleanupAction, err := m.generateContainerConfig(ctx, container, pod, restartCount, podIP, imageRef, podIPs, target)

	// PreCreateContainer, 尝试是否做cpu亲和及内存 numa, 现在只有 linux 平台做了具体实现.
	err = m.internalLifecycle.PreCreateContainer(pod, container, containerConfig)
	if err != nil {
		return s.Message(), ErrPreCreateHook
	}

	// 正式创建容器
	containerID, err := m.runtimeService.CreateContainer(ctx, podSandboxID, containerConfig, podSandboxConfig)
	if err != nil {
		return s.Message(), ErrCreateContainer
	}

	// 预先启动容器
	err = m.internalLifecycle.PreStartContainer(pod, container, containerID)
	if err != nil {
		return s.Message(), ErrPreStartHook
	}

	// 启动上面生成的容器
	err = m.runtimeService.StartContainer(ctx, containerID)
	if err != nil {
		return s.Message(), kubecontainer.ErrRunContainer
	}

	// 为容器的日志目录做软连接
	if _, err := m.osInterface.Stat(containerLog); !os.IsNotExist(err) {
		if err := m.osInterface.Symlink(containerLog, legacySymlink); err != nil {
			klog.ErrorS(err, "Failed to create legacy symbolic link", "path", legacySymlink,
				"containerID", containerID, "containerLogPath", containerLog)
		}
	}

	// 执行 post start hook
	if container.Lifecycle != nil && container.Lifecycle.PostStart != nil {
		kubeContainerID := kubecontainer.ContainerID{
			Type: m.runtimeName,
			ID:   containerID,
		}
		// 如果在容器启动后, 执行 post start hook 失败, 则干掉容器
		msg, handlerErr := m.runner.Run(ctx, kubeContainerID, pod, container, container.Lifecycle.PostStart)
		if handlerErr != nil {
			if err := m.killContainer(ctx, pod, kubeContainerID, container.Name, "FailedPostStartHook", reasonFailedPostStartHook, nil); err != nil {
			}
			return msg, ErrPostStartHook
		}
	}

	return "", nil
}
```

### 删除 pod 流程 (HandlePodRemoves)

删除 pod 的流程, 首先在 podManager 清理 pod, 然后调用 `podWorkers.UpdatePod` 方法来更新 Pod, 只是 updateType 为 `kubetypes.SyncPodKill`.

```go
func (kl *Kubelet) HandlePodRemoves(pods []*v1.Pod) {
	for _, pod := range pods {
		kl.podManager.DeletePod(pod)
		if kubetypes.IsMirrorPod(pod) {
			kl.handleMirrorPod(pod, start)
			continue
		}

		kl.deletePod(pod)
	}
}

func (kl *Kubelet) deletePod(pod *v1.Pod) error {
	if !kl.sourcesReady.AllReady() {
		return fmt.Errorf("skipping delete because sources aren't ready yet")
	}
	kl.podWorkers.UpdatePod(UpdatePodOptions{
		Pod:        pod,
		UpdateType: kubetypes.SyncPodKill,
	})
	return nil
}
```
