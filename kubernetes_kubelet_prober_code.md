# 源码分析 kubernetes kubelet prober 探针的实现原理

kubelet 使用 livenessProbe 存活探针来确定什么时候要重启容器, 使用 readinessProbe 就绪探针可以确认是否要把流量接入到 service 里, startupProbe 启动探针是为了避免在启动时间不可控时, 使用 liveness 探针探测失败, 造成重启的死循环的场景.

## 探针参数介绍

探针的参数介绍:

- initialDelaySeconds <integer>: 存活性探测延迟时长，即容器启动多久之后再开始第一次探测操作，显示为 delay 属性；默认为 0 秒，即容器启动后立刻便开始进行探测。
- timeoutSeconds <integer>: 存活性探测的超时时长，显示为 timeout 属性，默认为 1s，最小值也为 1s。
- periodSeconds <integer>: 存活性探测的频度，显示为 period 属性，默认为 10s, 最小值为 1s；过高的频率会对 Pod 对象带来较大的额外开销，而过低的频率又会使得对错误的反映不及时。
- successThreshold <integer>: 处于失败状态时，探测操作至少连续多少次的成功才被认为是通过检测，显示为 #success 属性，默认值为1，最小值也为1。
- failureThreshold <integer>: 处于成功状态时，探测操作至少连续多少次的失败才被视为是检测不通过，显示为 #failure 属性，默认值为 3，最小值为1。

## 源码分析

### 探针启动入口

实例化 kubelet 时, 顺便也会实例化 `prober` 探针管理器

```go
if kubeDeps.ProbeManager != nil {
	klet.probeManager = kubeDeps.ProbeManager
} else {
	klet.probeManager = prober.NewManager(
		klet.statusManager,
		klet.livenessManager,
		klet.readinessManager,
		klet.startupManager,
		klet.runner,
		kubeDeps.Recorder)
}
```

### 探针管理器

下面是 prober 管理器的数据结构组成:

`pkg/kubelet/prober/prober_manager.go`

```go
type probeKey struct {
	podUID        types.UID
	containerName string
	probeType     probeType
}

type manager struct {
	// 存放 worker, key 为 probeKey
	workers map[probeKey]*worker
	workerLock sync.RWMutex

	// 状态管理器
	statusManager status.Manager

	// 探针结果的封装
	readinessManager results.Manager
	livenessManager results.Manager
	startupManager results.Manager

	// 具体的prober的实现
	prober *prober

	// 启动时间, 为了后面做探针的 delay
	start time.Time
}
```

下面是 prober Manager 实现的接口. 后面分析下 `AddPod`, `StopLivenessAndStartup`, `RemovePod` 的实现.

```go
type Manager interface {
	// 为新 pod 增加探针
	AddPod(pod *v1.Pod)
	// 关闭 pod 时, 先停止 liveness 和 startup 探针
	StopLivenessAndStartup(pod *v1.Pod)
	// 关闭 pod 时, 删除其所有类型的探针
	RemovePod(pod *v1.Pod)
	// 清理 pods
	CleanupPods(desiredPods map[types.UID]sets.Empty)
	// 更新 pod 状态
	UpdatePodStatus(types.UID, *v1.PodStatus)
}
```

#### 为 Pod 添加探活

kubelet 在创建完容器后, 调用探针管理器的 `AddPod` 方法为 pod 开启探针. 每个探针都是一个 worker 协程. k8s node 节点默认允许最多创建 110 个Pod, 由 kubelet 的 `--max-pods` 参数控制, 开满三个探针也才几百个协程.

```go
func (m *manager) AddPod(pod *v1.Pod) {
	m.workerLock.Lock()
	defer m.workerLock.Unlock()

	// 组装 key
	key := probeKey{podUID: pod.UID}
	for _, c := range pod.Spec.Containers {
		key.containerName = c.Name

		// 如果有配置启动探针, 则创建
		if c.StartupProbe != nil {
			// 指定类型为 startup
			key.probeType = startup
			// 如果已经添加 key, 则跳过
			if _, ok := m.workers[key]; ok {
				return
			}
			// 为该 pod 的 startup 模式创建一个 worker 实例
			w := newWorker(m, startup, pod, c)
			m.workers[key] = w
			// 启动该 worker 实例
			go w.run()
		}
		// 如有配置 readiness 探针, 则创建
		if c.ReadinessProbe != nil {
			key.probeType = readiness
			if _, ok := m.workers[key]; ok {
				return
			}
			w := newWorker(m, readiness, pod, c)
			m.workers[key] = w
			go w.run()
		}

		// 如有配置 liveness 探针, 则创建
		if c.LivenessProbe != nil {
			...
		}
	}
}
```

#### 关闭 pod 探针

kubelet 在关闭 pod 时, 先调用 `StopLivenessAndStartup` 停止 liveness 和 startup 探针, 再进行 `killPod`, 之后再调用 `RemovePod` 来移除所有的探针.

kubelet 在关闭 pod 时对探针的操作:

`pkg/kubelet/kubelet.go`

```go
func (kl *Kubelet) syncTerminatingPod(_ context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, runningPod *kubecontainer.Pod, gracePeriod *int64, podStatusFn func(*v1.PodStatus)) error {

	// 只关闭 pod 的 liveness 和 startup 探针
	kl.probeManager.StopLivenessAndStartup(pod)

	// 给 pod 发送 SIGTERM 信号，超时未退出则强制 SIGKILL 杀掉容器.
	if err := kl.killPod(ctx, pod, p, gracePeriod); err != nil {
		return err
	}

	// 在探针管理器里删除该 pod 所有类型的探针
	kl.probeManager.RemovePod(pod)

	return nil
}
```

**StopLivenessAndStartup**

```go
func (m *manager) StopLivenessAndStartup(pod *v1.Pod) {
	m.workerLock.RLock()
	defer m.workerLock.RUnlock()

	key := probeKey{podUID: pod.UID}
	for _, c := range pod.Spec.Containers {
		key.containerName = c.Name
		for _, probeType := range [...]probeType{liveness, startup} {
			key.probeType = probeType
			if worker, ok := m.workers[key]; ok {
				worker.stop()
			}
		}
	}
}
```

**RemovePod**

```go
func (m *manager) RemovePod(pod *v1.Pod) {
	m.workerLock.RLock()
	defer m.workerLock.RUnlock()

	key := probeKey{podUID: pod.UID}
	for _, c := range pod.Spec.Containers {
		key.containerName = c.Name
		for _, probeType := range [...]probeType{readiness, liveness, startup} {
			key.probeType = probeType
			if worker, ok := m.workers[key]; ok {
				worker.stop()
			}
		}
	}
}
```

#### 清理探测器

kubelet 周期性执行 `HandlePodCleanups` 方法, 该方法会清理一波状态异常的容器, 同时也会清理探针.

```go
func (m *manager) CleanupPods(desiredPods map[types.UID]sets.Empty) {
	m.workerLock.RLock()
	defer m.workerLock.RUnlock()

	for key, worker := range m.workers {
		if _, ok := desiredPods[key.podUID]; !ok {
			worker.stop()
		}
	}
}
```

### prober worker 实现

开一个 ticker 定时器, 然后到期触发 `doProbe()`

```go
func (w *worker) run() {
	probeTickerPeriod := time.Duration(w.spec.PeriodSeconds) * time.Second
	...
	probeTicker := time.NewTicker(probeTickerPeriod)

probeLoop:
	for w.doProbe(ctx) {
		select {
		case <-w.stopCh:
			break probeLoop
		case <-probeTicker.C:
		}
	}
}
```

下面是具体探针检测的逻辑. 尝试去探测, 完成后进行次数累加, 如果探测的次数超过阈值, 则向 kubelet 传递状态更新.

```go
func (w *worker) doProbe(ctx context.Context) (keepGoing bool) {
	startTime := time.Now()

	// 如果 pod 还未被创建，或者 已经被删除会出现, ok 为 false 的情况.
	status, ok := w.probeManager.statusManager.GetPodStatus(w.pod.UID)
	if !ok {
		return true
	}

	// 如果 pod 失败或者退出, 则跳出
	if status.Phase == v1.PodFailed || status.Phase == v1.PodSucceeded {
		klog.V(3).InfoS("Pod is terminated, exiting probe worker",
			"pod", klog.KObj(w.pod), "phase", status.Phase)
		return false
	}

	c, ok := podutil.GetContainerStatus(status.ContainerStatuses, w.container.Name)
	if !ok || len(c.ContainerID) == 0 {
		return true // Wait for more information.
	}

	if w.containerID.String() != c.ContainerID {
		...
		w.resultsManager.Set(w.containerID, w.initialValue, w.pod)
		// 新容器起来时, 恢复探针检测
		w.onHold = false
	}

	// 旧容器因为不符合 liveness 规则被重启, 新容器还没起来时, 一直跳过探针检测.
	if w.onHold {
		return true
	}

	// 如果在 graceful shutdown 阶段
	if w.pod.ObjectMeta.DeletionTimestamp != nil && (w.probeType == liveness || w.probeType == startup) {
		w.resultsManager.Set(w.containerID, results.Success, w.pod)
		return false
	}

	// 探测要在 initialDelay 延迟参数后才启用
	if int32(time.Since(c.State.Running.StartedAt.Time).Seconds()) < w.spec.InitialDelaySeconds {
		return true
	}

	// 真正的执行探活逻辑
	result, err := w.probeManager.prober.probe(ctx, w.probeType, w.pod, status, w.container, w.containerID)
	if err != nil {
		return true
	}

	// 状态相同, 则递增加一
	// 状态不同, 则直接重置, 重新累加.
	if w.lastResult == result {
		w.resultRun++
	} else {
		w.lastResult = result
		w.resultRun = 1
	}

	// 探测失败了, 但次数在阈值以内, 不用传递 pod 状态
	// 或者探测成功了, 且次数在阈值以内, 不用传递 pod 状态
	if (result == results.Failure && w.resultRun < int(w.spec.FailureThreshold)) ||
		(result == results.Success && w.resultRun < int(w.spec.SuccessThreshold)) {
		return true
	}

	// 像 kubelet 传递当前 pod 变更状态, kubelet 根据状态选择策略.
	w.resultsManager.Set(w.containerID, result, w.pod)

	// 重置计数 
	if (w.probeType == liveness || w.probeType == startup) && result == results.Failure {
		w.onHold = true
		w.resultRun = 0
	}

	return true
}
```

### 真正的探测逻辑

重试三遍执行探测逻辑, 根据探针方法不同启动不同的探针方法. 

- exec 探针的 exit code 返回 0 时, 认为探测成功. 
- http 探针的 resp code 返回 2xx 和 3xx 时, 认为探测成功.
- tcp  探测可以完成建连, 也就是三次握手, 则认为探测成功.
- grpc 探测需要业务服务集成 `grpchealth` 包, 正常返回 'SERVING' 状态, 则认为探测成功.

```go
const maxProbeRetries = 3

func (pb *prober) runProbeWithRetries(ctx context.Context, probeType probeType, p *v1.Probe, pod *v1.Pod, status v1.PodStatus, container v1.Container, containerID kubecontainer.ContainerID, retries int) (probe.Result, string, error) {
	var err error
	var result probe.Result
	var output string
	for i := 0; i < retries; i++ {
		result, output, err = pb.runProbe(ctx, probeType, p, pod, status, container, containerID)
		if err == nil {
			return result, output, nil
		}
	}
	return result, output, err
}

func (pb *prober) runProbe(ctx context.Context, probeType probeType, p *v1.Probe, pod *v1.Pod, status v1.PodStatus, container v1.Container, containerID kubecontainer.ContainerID) (probe.Result, string, error) {
	// 超时时间
	timeout := time.Duration(p.TimeoutSeconds) * time.Second

	// 执行 exec 类型探针
	if p.Exec != nil {
		command := kubecontainer.ExpandContainerCommandOnlyStatic(p.Exec.Command, container.Env)
		return pb.exec.Probe(pb.newExecInContainer(ctx, container, containerID, command, timeout))
	}
	// 执行 httpGet 类型探针
	if p.HTTPGet != nil {
		req, err := httpprobe.NewRequestForHTTPGetAction(p.HTTPGet, &container, status.PodIP, "probe")
		if err != nil {
			return probe.Unknown, "", err
		}
		...
		return pb.http.Probe(req, timeout)
	}

	// 执行 tcp 类型探针
	if p.TCPSocket != nil {
		...
		return pb.tcp.Probe(host, port, timeout)
	}

	// 执行 grpc 类型探针
	if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.GRPCContainerProbe) && p.GRPC != nil {
		...
		return pb.grpc.Probe(host, service, int(p.GRPC.Port), timeout)
	}

	return probe.Unknown, "", fmt.Errorf("missing probe handler for %s:%s", format.Pod(pod), container.Name)
}
```

### 传递状态到 kubelet syncLoop 调度循环

prober 探测到 pod 异常且符合阈值后, 调用 resultsManager.Set 传递状态, 其实就是往 updates 管道发送事件.

```go
func (m *manager) Set(id kubecontainer.ContainerID, result Result, pod *v1.Pod) {
	if m.setInternal(id, result) {
		m.updates <- Update{id, result, pod.UID}
	}
}
```

kubelet 主循环调度 syncLoop 里的 syncLoopIteration 会监听该 updates 管道. 当有事件通知时, 先更改 statusManager 状态并通过 handleProbeSync 执行 podWorkers.UpdatePod 容器更新逻辑.

```go
func (kl *Kubelet) syncLoopIteration(...) bool {
	select {
	case u, open := <-configCh:
		...
	case e := <-plegCh:
		...
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
		}
	case update := <-kl.readinessManager.Updates():
		ready := update.Result == proberesults.Success
		kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)

		status := ""
		if ready {
			status = "ready"
		}
		handleProbeSync(kl, update, handler, "readiness", status)
	case update := <-kl.startupManager.Updates():
		started := update.Result == proberesults.Success
		kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)

		status := "unhealthy"
		if started {
			status = "started"
		}
		handleProbeSync(kl, update, handler, "startup", status)
	}
	return true
}
```