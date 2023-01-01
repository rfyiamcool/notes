# 源码分析 kubernetes endpoints controller 的实现原理

## 源码分析

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212131143392.png)

`基于 k8s v1.27.0 源码分析`

### 启动入口

传入 pods, services, endpoints 三种资源的 informer, 实例化 `EndpointsController` 端点控制器对象.

```go
func startEndpointController(ctx context.Context, controllerCtx ControllerContext) (controller.Interface, bool, error) {
	go endpointcontroller.NewEndpointController(
		controllerCtx.InformerFactory.Core().V1().Pods(),
		controllerCtx.InformerFactory.Core().V1().Services(),
		controllerCtx.InformerFactory.Core().V1().Endpoints(),
	).Run(ctx, int(controllerCtx.ComponentConfig.EndpointController.ConcurrentEndpointSyncs))
	return nil, true, nil
}
```

实例化 `EndpointController` 控制器, 内部在 services, pods, endpoints 三种资源的 informer 里, 注册 EventHandler 事件回调处理方法.

```go
func NewEndpointController(podInformer coreinformers.PodInformer, serviceInformer coreinformers.ServiceInformer,
	endpointsInformer coreinformers.EndpointsInformer, client clientset.Interface, endpointUpdatesBatchPeriod time.Duration) *Controller {
	e := &Controller{
		client:           client,
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "endpoint"),
		workerLoopPeriod: time.Second,
	}

	// 监听 services informer, 并注册事件方法
	serviceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: e.onServiceUpdate,
		UpdateFunc: func(old, cur interface{}) {
			e.onServiceUpdate(cur)
		},
		DeleteFunc: e.onServiceDelete,
	})
	e.serviceLister = serviceInformer.Lister()
	e.servicesSynced = serviceInformer.Informer().HasSynced

	// 监听 pods informer, 并注册事件方法
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    e.addPod,
		UpdateFunc: e.updatePod,
		DeleteFunc: e.deletePod,
	})
	e.podLister = podInformer.Lister()
	e.podsSynced = podInformer.Informer().HasSynced

	// 监听 endpoints informer, 并注册事件方法
	endpointsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: e.onEndpointsDelete,
	})
	e.endpointsLister = endpointsInformer.Lister()
	e.endpointsSynced = endpointsInformer.Informer().HasSynced
	return e
}
```

### 启动控制器

Run() 用来启动控制器, 先同步各个资源的数据到本地, 然后启动 workers 数量的协程去启动 worker. workers 的数量由 `--concurrent-endpoint-syncs` 启动参数指定, 默认为 5 个.

```go
func (e *Controller) Run(ctx context.Context, workers int) {
	// 等待 pods, services, endpoints 资源同步完成
	if !cache.WaitForNamedCacheSync("endpoint", ctx.Done(), e.podsSynced, e.servicesSynced, e.endpointsSynced) {
		return
	}

	// 启动 workers 数量的协程来处理 endpoints
	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, e.worker, e.workerLoopPeriod)
	}

	go func() {
		e.checkLeftoverEndpoints()
	}()

	<-ctx.Done()
}
```

启动 worker 的协程逻辑还是整洁的, 不断的从队列中获取任务, 然后调用 syncService 来同步处理.

```go
func (e *Controller) worker(ctx context.Context) {
	for e.processNextWorkItem(ctx) {
	}
}

func (e *Controller) processNextWorkItem(ctx context.Context) bool {
	// 无任务时, 陷入等待
	eKey, quit := e.queue.Get()
	if quit {
		return false
	}
	defer e.queue.Done(eKey)

	err := e.syncService(ctx, eKey.(string))
	...
	return true
}
```

### infomer eventHandler

实例化 `endpointsController` 时会对 pod, service, endpoint 资源进行注册 eventHandler 并监听.

下面分析这三种资源对应的 eventHandler 逻辑.

#### service eventhandler

增删减 serviceSelectorCache 缓存数据, 并把 key 塞入到队列.

```go
func (e *Controller) onServiceUpdate(obj interface{}) {
	key, err := controller.KeyFunc(obj)
	if err != nil {
		return
	}

	_ = e.serviceSelectorCache.Update(key, obj.(*v1.Service).Spec.Selector)
	e.queue.Add(key)
}

func (e *Controller) onServiceDelete(obj interface{}) {
	key, err := controller.KeyFunc(obj)
	if err != nil {
		return
	}

	e.serviceSelectorCache.Delete(key)
	e.queue.Add(key)
}
```

`serviceSelectorCache` 用来管理各个 service 的 labels.Selector 缓存, 后面 pod eventHandler 里通过该 cache 获取 pod labels 关联的 services 列表. 

```go
type ServiceSelectorCache struct {
	lock  sync.RWMutex
	cache map[string]labels.Selector
}

func (sc *ServiceSelectorCache) Get(key string) (labels.Selector, bool) {
	...
}

func (sc *ServiceSelectorCache) Update(key string, rawSelector map[string]string) labels.Selector {
	...
}

func (sc *ServiceSelectorCache) Delete(key string) {
	...
}

// 遍历并查找适配该pod labels 的 services 集合
func (sc *ServiceSelectorCache) GetPodServiceMemberships(serviceLister v1listers.ServiceLister, pod *v1.Pod) (sets.String, error) {
	set := sets.String{}
	services, err := serviceLister.Services(pod.Namespace).List(labels.Everything())
	if err != nil {
		return set, err
	}

	var selector labels.Selector
	for _, service := range services {
		if service.Spec.Selector == nil {
			continue
		}
		key, err := controller.KeyFunc(service)
		if err != nil {
			return nil, err
		}
		if v, ok := sc.Get(key); ok {
			selector = v
		} else {
			selector = sc.Update(key, service.Spec.Selector)
		}

		if selector.Matches(labels.Set(pod.Labels)) {
			set.Insert(key)
		}
	}
	return set, nil
}
```

#### pod eventHandler

`addPod` 需要通过 pod 获取关联的 services 对象列表, 然后遍历 servcies 列表把每个 service 都通过延迟入队的方法入队. 不同的 service 通过标签可以关联同一个 pod.

像 updatePod 和 deletePod 逻辑相似, 不再复述.

```go
func (e *Controller) addPod(obj interface{}) {
	pod := obj.(*v1.Pod)
	services, err := e.serviceSelectorCache.GetPodServiceMemberships(e.serviceLister, pod)
	if err != nil {
		return
	}
	for key := range services {
		e.queue.AddAfter(key, e.endpointUpdatesBatchPeriod)
	}
}

func (e *Controller) updatePod(old, cur interface{}) {
	services := endpointutil.GetServicesToUpdateOnPodChange(e.serviceLister, e.serviceSelectorCache, old, cur)
	for key := range services {
		e.queue.AddAfter(key, e.endpointUpdatesBatchPeriod)
	}
}

func (e *Controller) deletePod(obj interface{}) {
	pod := endpointutil.GetPodFromDeleteAction(obj)
	if pod != nil {
		e.addPod(pod)
	}
}
```

#### endpoints eventHandler

只在 `endpointsInformer` 里注册了 DeleteFunc 处理方法.

```go
func (e *Controller) onEndpointsDelete(obj interface{}) {
	key, err := controller.KeyFunc(obj)
	if err != nil {
		return
	}
	e.queue.Add(key)
}
```

### 核心处理函数 syncService

#### 主处理流程

`syncService` 是 endpoints controller 控制里最核心的方法, 逻辑相对其他 controller 控制器还是要简单不少. 

1. 获取 service 对象, 如果没找到该对象, 则尝试向 apiserver 发起删除 endpoints 请求.
2. 根据 labels 获取 service 对应的 pods 对象
3. 定义 subset 子集对象, 遍历 pods 列表生成 EndpointSubset 对象, 并合并到 subset 里.
4. 尝试获取 endpoints 对象, 没有则执行创建 endpoints 操作, 有则执行更新操作.

```go
// xiaorui.cc
func (e *Controller) syncService(ctx context.Context, key string) error {
	// 通过 key 拆解 namespace 和 service name 字段.
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}

	// 获取 service 对象
	service, err := e.serviceLister.Services(namespace).Get(name)
	if err != nil {
		// 其他错误直接抛出异常
		if !errors.IsNotFound(err) {
			return err
		}

		// 向 apiserver 发起删除 service 的 endpoints 的请求
		err = e.client.CoreV1().Endpoints(namespace).Delete(ctx, name, metav1.DeleteOptions{})
		if err != nil && !errors.IsNotFound(err) {
			return err
		}
		return nil
	}

	// 如果 .spec.selector 为空, 则没有关联对象, 可直接返回.
	if service.Spec.Selector == nil {
		return nil
	}

	// 获取 service 对应 labels 的 pods 对象集合.
	pods, err := e.podLister.Pods(service.Namespace).List(labels.Set(service.Spec.Selector).AsSelectorPreValidated())
	if err != nil {
		return err
	}

	subsets := []v1.EndpointSubset{}
	var totalReadyEps int
	var totalNotReadyEps int

	for _, pod := range pods {
		// 如果 pod 终止状态, 或 podIP 为空, 或已经标记删除, 则直接跳过该 pod. 
		if !endpointutil.ShouldPodBeInEndpoints(pod, service.Spec.PublishNotReadyAddresses) {
			continue
		}

		// 实例化 v1.EndpointAddress 对象
		ep, err := podToEndpointAddressForService(service, pod)
		if err != nil {
			continue
		}

		epa := *ep
		if len(service.Spec.Ports) == 0 {
			// 在 headless 模式下, 构建一个 v1.EndpointSubset 对象, append 到 subsets 子集里.
			if service.Spec.ClusterIP == api.ClusterIPNone {
				subsets, totalReadyEps, totalNotReadyEps = addEndpointSubset(subsets, pod, epa, nil, service.Spec.PublishNotReadyAddresses)
			}
		} else {
			for i := range service.Spec.Ports {
				servicePort := &service.Spec.Ports[i]
				portNum, err := podutil.FindPort(pod, servicePort)
				// 创建一个 endpoint port 对象
				epp := endpointPortFromServicePort(servicePort, portNum)

				// 创建 v1.EndpointSubset 对象, 并 append subsets 子集里.
				// 返回 readyEps 就绪的 pod 数量和 notReadyEps 没就绪的 pod 数量
				var readyEps, notReadyEps int
				subsets, readyEps, notReadyEps = addEndpointSubset(subsets, pod, epa, epp, service.Spec.PublishNotReadyAddresses)
				// 累加就绪的数量
				totalReadyEps = totalReadyEps + readyEps
				// 累加还没就绪的数量
				totalNotReadyEps = totalNotReadyEps + notReadyEps
			}
		}
	}
	subsets = endpoints.RepackSubsets(subsets)

	// 从 informer 里尝试获取当前的 endpoints 对象
	currentEndpoints, err := e.endpointsLister.Endpoints(service.Namespace).Get(service.Name)
	if err != nil {
		if errors.IsNotFound(err) {
			// 如果没有则实例化一个 endpoints 对象
			currentEndpoints = &v1.Endpoints{
				ObjectMeta: metav1.ObjectMeta{
					Name:   service.Name,
					Labels: service.Labels,
				},
			}
		}
	}

	// 如果版本为空, 则需要创建
	createEndpoints := len(currentEndpoints.ResourceVersion) == 0

	newEndpoints.Subsets = subsets
	newEndpoints.Labels = service.Labels

	if createEndpoints {
		// 如果没创建过, 则创建 endpoints 对象
		_, err = e.client.CoreV1().Endpoints(service.Namespace).Create(ctx, newEndpoints, metav1.CreateOptions{})
	} else {
		// 已经存在 ep 对象, 则需要更新
		_, err = e.client.CoreV1().Endpoints(service.Namespace).Update(ctx, newEndpoints, metav1.UpdateOptions{})
	}
	...
	return nil
}
```

#### 哪些 pod 放到 endpoints 里 ?

`ShouldPodBeInEndpoints` 方法里定义了哪些 pod 会被放到 subset 里, 也就是会被放到 endpoints 集合里, 只有为 true 才会处理.

```go
func ShouldPodBeInEndpoints(pod *v1.Pod, includeTerminating bool) bool {
	// "Terminal" describes when a Pod is complete (in a succeeded or failed phase).
	// This is distinct from the "Terminating" condition which represents when a Pod
	// is being terminated (metadata.deletionTimestamp is non nil).
	// xiaorui.cc
	if podutil.IsPodTerminal(pod) {
		return false
	}

	if len(pod.Status.PodIP) == 0 && len(pod.Status.PodIPs) == 0 {
		return false
	}

	if !includeTerminating && pod.DeletionTimestamp != nil {
		return false
	}

	return true
}
```