# 源码分析 kubernetes apisix ingress controller 控制器的实现原理

apisix ingress 相比 nginx ingress 来说有太多的优势, apisix 可以做到全动态配置加载, 而 nginx ingres 只能做到部分动态加载. 另外 apisix 有更丰富的插件可以使用, 不像 nginx ingress 不易扩展插件.

apisix ingress controller 相比 nginx ingrss controller 实现要简单一些. apisix ingress controller 只需监听 k8s 内置和 crd 资源, 当有事件变更时, 则向 apisix admin api 发起配置变更请求即可. 到此 apisix ingress 控制面的流程完成了, 后面 apisix admin 把收到配置写到 etcd 里. 其他作为数据面的 apisix 监听到 etcd 的配置变更后, 进行动态配置加载. 

需要关注的是, nginx ingress controller 是不区分控制面和数据面, 每个 nginx ingress 实例不仅是 controller 控制器角色, 而且也是 nginx server 角色. 当配置发生变更时, nginx ingress 会对本容器的 nginx 进程进行维护, 对于 endpoints 变更则可以配合 `balancer_by_lua` 进行 upstream 配置的动态更新.

如果对 nginx ingress controller 实现原理感兴趣, 则可以点击下面连接. 

- [源码分析 kubernetes ingress nginx controller 控制器的实现原理](https://github.com/rfyiamcool/notes/blob/main/kubernetes_ingress_nginx_controller_code.md)
- [源码分析 kubernetes ingress nginx 动态更新的实现原理](https://github.com/rfyiamcool/notes/blob/main/kubernetes_ingress_nginx_controller_dynamic_update_code.md)

## 项目介绍

apisix ingress controller 项目地址.

[https://github.com/apache/apisix-ingress-controller](https://github.com/apache/apisix-ingress-controller)

**apisix ingress controller 架构图**

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301102333471.png)

**apisix ingress controller 简化流程图**

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/Jietu20230110-230003.jpg)

## 入口

废话不多说, apisix ingress controller 的启动是这样推进的, main/cobra/flag 没什么可说的.

```c
main.go -> cmd/ingress/ingress.go -> pkg/providers/controller.go
```

## 启动控制器

apisix ingress controller 也是支持高可用性的, 其实现方法跟 k8s 中其他 controller 一样, 都是依赖 leaderelection 选举实现的, 拿到 leader 的实例可以进行同步操作. 

```go
func (c *Controller) Run(stop chan struct{}) error {
	rootCtx, rootCancel := context.WithCancel(context.Background())
	defer rootCancel()
	go func() {
		<-stop
		rootCancel()
	}()

	go func() {
		// 这个 apiserver 没什么东西, 都是些 pprof 和一些查询数据的接口.
		if err := c.apiServer.Run(rootCtx.Done()); err != nil {
			log.Errorf("failed to launch API Server: %s", err)
		}
	}()

	// 定义锁结构
	lock := &resourcelock.LeaseLock{
		LeaseMeta: metav1.ObjectMeta{
			Namespace: c.namespace,
			Name:      c.cfg.Kubernetes.ElectionID,
		},
		Client: c.kubeClient.Client.CoordinationV1(),
	}

	// 定义 lederelection 选举的配置
	cfg := leaderelection.LeaderElectionConfig{
		Lock:          lock,
		LeaseDuration: 15 * time.Second,
		RenewDeadline: 5 * time.Second,
		RetryPeriod:   2 * time.Second,
		Callbacks: leaderelection.LeaderCallbacks{
			// 拿到 leader 启动 run() 方法.
			OnStartedLeading: c.run,
			OnNewLeader: func(identity string) {
				...
			},
			OnStoppedLeading: func() {
			},
		},
		ReleaseOnCancel: true,
		Name:            "ingress-apisix",
	}
	elector, err := leaderelection.NewLeaderElector(cfg)
	...

election:
	curCtx, cancel := context.WithCancel(rootCtx)

	// 运行发生异常, 则调用 cancel 退出
	c.leaderContextCancelFunc = cancel

	// 启动选举, 拿到 leader 时调用 run() 方法.
	elector.Run(curCtx)

	select {
	case <-rootCtx.Done():
		return nil
	default:
		goto election
	}
}
```

实例化一波 provider, 然后启动这些 provider.

```go
func (c *Controller) run(ctx context.Context) {
	var cancelFunc context.CancelFunc
	ctx, cancelFunc = context.WithCancel(ctx)
	defer cancelFunc()

	// give up leader
	defer c.leaderContextCancelFunc()

	// 把访问 apisix 的一些配置放到 opts 里.
	clusterOpts := &apisix.ClusterOptions{
		AdminAPIVersion:  c.cfg.APISIX.AdminAPIVersion,
		Name:             c.cfg.APISIX.DefaultClusterName,
		AdminKey:         c.cfg.APISIX.DefaultClusterAdminKey,
		BaseURL:          c.cfg.APISIX.DefaultClusterBaseURL,
		MetricsCollector: c.MetricsCollector,
	}
	err := c.apisix.AddCluster(ctx, clusterOpts)

	// 实例化 informers 合集
	c.informers = c.initSharedInformers()
	common := &providertypes.Common{
		ControllerNamespace: c.namespace,
		ListerInformer:      c.informers,
		Config:              c.cfg,
		APISIX:              c.apisix,
		KubeClient:          c.kubeClient,
		MetricsCollector:    c.MetricsCollector,
		Recorder:            c.recorder,
	}

	// 实例化 namespace provider
	c.namespaceProvider, err = namespace.NewWatchingNamespaceProvider(ctx, c.kubeClient, c.cfg)
	if err != nil {
		ctx.Done()
		return
	}

	// 实例化 namespace provider
	c.podProvider, err = pod.NewProvider(common, c.namespaceProvider)
	if err != nil {
		ctx.Done()
		return
	}

	// 实例化结构转换对象
	c.translator = translation.NewTranslator(&translation.TranslatorOptions{
		APIVersion:           c.cfg.Kubernetes.APIVersion,
		EndpointLister:       c.informers.EpLister,
		ServiceLister:        c.informers.SvcLister,
		SecretLister:         c.informers.SecretLister,
		PodLister:            c.informers.PodLister,
		ApisixUpstreamLister: c.informers.ApisixUpstreamLister,
		PodProvider:          c.podProvider,
	})

	// 实例化
	c.apisixProvider, c.apisixTranslator, err = apisixprovider.NewProvider(common, c.namespaceProvider, c.translator)
	if err != nil {
		ctx.Done()
		return
	}

	// 实例化 ingress provider
	c.ingressProvider, err = ingressprovider.NewProvider(common, c.namespaceProvider, c.translator, c.apisixTranslator)
	if err != nil {
		ctx.Done()
		return
	}

	// 实例化 k8s sprovider
	c.kubeProvider, err = k8s.NewProvider(common, c.translator, c.namespaceProvider, c.apisixProvider, c.ingressProvider)
	if err != nil {
		ctx.Done()
		return
	}

	// 开启 gateway 处理, 默认不开启.
	if c.cfg.Kubernetes.EnableGatewayAPI {
		c.gatewayProvider, err = gateway.NewGatewayProvider(&gateway.ProviderOptions{
			Cfg:               c.cfg,
			APISIX:            c.apisix,
			APISIXClusterName: c.cfg.APISIX.DefaultClusterName,
			KubeTranslator:    c.translator,
			RestConfig:        nil,
			KubeClient:        c.kubeClient.Client,
			MetricsCollector:  c.MetricsCollector,
			NamespaceProvider: c.namespaceProvider,
		})
		if err != nil {
			ctx.Done()
			return
		}
	}

	...

	e := utils.ParallelExecutor{}

	e.Add(func() {
		// 启动 namespace provider
		c.namespaceProvider.Run(ctx)
	})

	e.Add(func() {
		// 启动 k8s provider
		c.kubeProvider.Run(ctx)
	})

	e.Add(func() {
		// 启动 apisix provider
		c.apisixProvider.Run(ctx)
	})

	e.Add(func() {
		// 启动 ingress provider
		c.ingressProvider.Run(ctx)
	})

	if c.cfg.Kubernetes.EnableGatewayAPI {
		e.Add(func() {
			// 启动 gateway provider
			c.gatewayProvider.Run(ctx)
		})
	}

	<-ctx.Done()
	e.Wait()

	if len(e.Errors()) > 0 {
		log.Error("Start failed, abort...")
		cancelFunc()
	}
}
```

## ingress provider

ingress provider 简化后的流程.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301102308817.png)

### 实例化 ingressController

实例化 `ingressController` 对象, 内部会向 ingress informer 注册 eventHandler 方法.

```go
func newIngressController(common *ingressCommon) *ingressController {
	c := &ingressController{
		workqueue: workqueue.NewNamedRateLimitingQueue(workqueue.NewItemFastSlowRateLimiter(1*time.Second, 60*time.Second, 5), "ingress"),
		workers:   1,
	}

	c.IngressInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    c.onAdd,
		UpdateFunc: c.onUpdate,
		DeleteFunc: c.OnDelete,
	})
	return c
}
```

下面是 ingress informer 注册的 eventHandler 事件方法. 当 informer 触发增删改回调时, 往 workqueue 插入对应类型的事件.

```go
// informer 添加调用
func (c *ingressController) onAdd(obj interface{}) {
	key, err := cache.MetaNamespaceKeyFunc(obj)

	// 过滤不合法的 namespace
	if !c.namespaceProvider.IsWatchingNamespace(key) {
		return
	}

	// 转换 ingress 结构 
	ing := kube.MustNewIngress(obj)

	// 向 workqueue 插入添加类型的 types.Event.
	c.workqueue.Add(&types.Event{
		Type: types.EventAdd,
		Object: kube.IngressEvent{
			Key:          key,
			GroupVersion: ing.GroupVersion(),
		},
	})

}

// informer 更新调用
func (c *ingressController) onUpdate(oldObj, newObj interface{}) {
	// 旧对象
	prev := kube.MustNewIngress(oldObj)

	// 新对象
	curr := kube.MustNewIngress(newObj)

	// 过滤异常
	if prev.ResourceVersion() >= curr.ResourceVersion() {
		return
	}

	key, err := cache.MetaNamespaceKeyFunc(newObj)
	if err != nil {
		return
	}

	// 过滤不合法的 namespace
	if !c.namespaceProvider.IsWatchingNamespace(key) {
		return
	}


	// 向 workqueue 插入更新的事件
	c.workqueue.Add(&types.Event{
		Type: types.EventUpdate,
		Object: kube.IngressEvent{
			Key:          key,
			GroupVersion: curr.GroupVersion(),
			OldObject:    prev,
		},
	})
}

// informer 删除调用
func (c *ingressController) OnDelete(obj interface{}) {
	ing, err := kube.NewIngress(obj)
	if err != nil {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			return
		}
		ing = kube.MustNewIngress(tombstone)
	}

	key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
	if err != nil {
		return
	}
	if !c.namespaceProvider.IsWatchingNamespace(key) {
		return
	}

	// 向 workqueue 插入删除的事件
	c.workqueue.Add(&types.Event{
		Type: types.EventDelete,
		Object: kube.IngressEvent{
			Key:          key,
			GroupVersion: ing.GroupVersion(),
		},
		Tombstone: ing,
	})
}
```

### 启动 ingress controller

只启动一个协程去消费 workqueue 队列, 使用 sync 处理获取的 types.Event.

```go
func (c *ingressController) run(ctx context.Context) {
	defer c.workqueue.ShutDown()

	// workers 为 1, 只启动一个协程.
	for i := 0; i < c.workers; i++ {
		go c.runWorker(ctx)
	}
	<-ctx.Done()
}

func (c *ingressController) runWorker(ctx context.Context) {
	for {
		obj, quit := c.workqueue.Get()
		if quit {
			return
		}
		err := c.sync(ctx, obj.(*types.Event))
		c.workqueue.Done(obj)
		c.handleSyncErr(obj, err)
	}
}
```

### 核心 sync 同步逻辑

```go
func (c *ingressController) sync(ctx context.Context, ev *types.Event) error {
	ingEv := ev.Object.(kube.IngressEvent)
	namespace, name, err := cache.SplitMetaNamespaceKey(ingEv.Key)
	if err != nil {
		return err
	}

	// 转换 ingress 的格式为 kube.Ingress 结构.
	var ing kube.Ingress
	switch ingEv.GroupVersion {
	case kube.IngressV1:
		ing, err = c.IngressLister.V1(namespace, name)
	case kube.IngressV1beta1:
		ing, err = c.IngressLister.V1beta1(namespace, name)
	case kube.IngressExtensionsV1beta1:
		ing, err = c.IngressLister.ExtensionsV1beta1(namespace, name)
	default:
		...
	}

	// 当事件类型为删除时, 从 Tombstone 字段中获取 ingress 对象.
	if ev.Type == types.EventDelete {
		if ing != nil {
			return nil
		}
		ing = ev.Tombstone.(kube.Ingress)
	}

	// 把 ingress 结构体转换成 translation.TranslateContext 结构.
	tctx, err := c.translator.TranslateIngress(ing)
	if err != nil {
		return err
	}

	// 填充结构
	m := &utils.Manifest{
		SSLs:          tctx.SSL,
		Routes:        tctx.Routes,
		Upstreams:     tctx.Upstreams,
		PluginConfigs: tctx.PluginConfigs,
	}

	// 增改删三个对象, 根据事件的不同, 只会用一个.
	var (
		added   *utils.Manifest
		updated *utils.Manifest
		deleted *utils.Manifest
	)

	if ev.Type == types.EventDelete {
		deleted = m
	} else if ev.Type == types.EventAdd {
		added = m
	} else {
		// 如果是update 事件类型, 需要转换下旧资源对象.
		oldCtx, err := c.translator.TranslateOldIngress(ingEv.OldObject)
		if err != nil {
			return err
		}
		om := &utils.Manifest{
			Routes:        oldCtx.Routes,
			Upstreams:     oldCtx.Upstreams,
			SSLs:          oldCtx.SSL,
			PluginConfigs: oldCtx.PluginConfigs,
		}
		// 通过 diff 计算有变动的配置部分
		added, updated, deleted = m.Diff(om)
	}
	
	// 调用 SyncManifests 同步配置, 该函数内部会处理 added, updated, deleted 不为空的配置.
	if err := c.SyncManifests(ctx, added, updated, deleted); err != nil {
		return err
	}
	return nil
}
```

### 真正操作 apisix 的方法

`SyncManifests` 用来真正处理 apisix 的入口, 其内部会通过判断 `added/updated/deleted` 对象是否为 nil, 决定最后走 create, update, delete 哪一步.

```go
func SyncManifests(ctx context.Context, apisix apisix.APISIX, clusterName string, added, updated, deleted *Manifest) error {
	if added != nil {
		for _, ssl := range added.SSLs {
			if _, err := apisix.Cluster(clusterName).SSL().Create(ctx, ssl); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		for _, u := range added.Upstreams {
			if _, err := apisix.Cluster(clusterName).Upstream().Create(ctx, u); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		for _, pc := range added.PluginConfigs {
			if _, err := apisix.Cluster(clusterName).PluginConfig().Create(ctx, pc); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		for _, r := range added.Routes {
			if _, err := apisix.Cluster(clusterName).Route().Create(ctx, r); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		...
	}
	if updated != nil {
		for _, ssl := range updated.SSLs {
			if _, err := apisix.Cluster(clusterName).SSL().Update(ctx, ssl); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		for _, r := range updated.Upstreams {
			if _, err := apisix.Cluster(clusterName).Upstream().Update(ctx, r); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		for _, pc := range updated.PluginConfigs {
			if _, err := apisix.Cluster(clusterName).PluginConfig().Update(ctx, pc); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		for _, r := range updated.Routes {
			if _, err := apisix.Cluster(clusterName).Route().Update(ctx, r); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		...
	}
	if deleted != nil {
		...
		...
	}
	if merr != nil {
		return merr
	}
	return nil
}
```

`SyncManifests` 涉及到的方法过多, 其实内部原理都大同小异, 大概流程把传入的结构体序列化为 bytes, 按照参数拼凑访问地址, 根据行为使用不同的 http method 进行请求, 访问完成后需要更新下缓存.

这里拿创建 apisix route 路由对象来说, 其他的 apisix 配置项的同步原理跟 route 的类似.

代码位置: `pkg/apisix/route.go`

```go
func (r *routeClient) Create(ctx context.Context, obj *v1.Route) (*v1.Route, error) {
	// 序列化 route 逻辑
	data, err := json.Marshal(obj)
	if err != nil {
		return nil, err
	}

	// 拼装 url 逻辑, 格式为 /apisix/admin/routes/{id}
	url := r.url + "/" + obj.ID

	// 通过 http put 来创建 route 对象
	resp, err := r.cluster.createResource(ctx, url, "route", data)
	if err != nil {
		return nil, err
	}

	route, err := resp.route()
	if err != nil {
		return nil, err
	}

	// 往缓冲写入 route 路由对象
	if err := r.cluster.cache.InsertRoute(route); err != nil {
		return nil, err
	}
	return route, nil
}
```

## k8s provider

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301102305782.png)

apisix ingress controller 内部实现了四个 k8s 内置资源的 provider 逻辑, 分别是 configmap, endpoint, namespace, pod 资源类型. 接口是 provider, 其内部的命名还是叫 controller 控制器.

由于篇幅原因, 这里只说下最关键的 endpoint provider, 其他的 provider 不做深入的分析.

**endpoint provider**

这里关于 endpoints 处理逻辑其实 nginx ingress 一样的, 也是为了同步 upstream 转发配置, apisix 也同样使用 endpoint 资源来处理 apisix 关联对象的 upstream 配置. 

**configmap provider**

向 apisix 管理接口同步 `PluginMetadatas` 配置.

**pod provider**

只是在本地构建了一个缓存, key 为 ip.

**namespace provider**

维护 `watchingNamespaces` 缓存, 当收到 namespace informer 事件时, 对该缓存进行增改操作. 

这个有 namespace 什么用呢? 

通过阅读 ingress apisix 代码得知, 各种资源在处理前都会判断当前对象的 ns 是否有效, 在缓存即为有效.

### 实例化 endpoints

实例化 endpointsController 控制器, 内部会通过 informer 注册事件方法并监听 endpoints 资源变化.

```go
func newEndpointsController(base *baseEndpointController, namespaceProvider namespace.WatchingNamespaceProvider) *endpointsController {
	ctl := &endpointsController{
		baseEndpointController: base,

		workqueue: workqueue.NewNamedRateLimitingQueue(workqueue.NewItemFastSlowRateLimiter(1*time.Second, 60*time.Second, 5), "endpoints"),
		workers:   1,

		namespaceProvider: namespaceProvider,

		epLister:   base.EpLister,
		epInformer: base.EpInformer,
	}

	ctl.epInformer.AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    ctl.onAdd,
			UpdateFunc: ctl.onUpdate,
			DeleteFunc: ctl.onDelete,
		},
	)

	return ctl
}
```

### 向 informer 注册的 eventHandler

向 informer 注册的 eventHandler 方法实现比较简单, 往 workqueue 传递 types.Evnet 事件即可.

```go
func (c *endpointsController) onAdd(obj interface{}) {
	key, err := cache.MetaNamespaceKeyFunc(obj)
	if err != nil {
		return
	}

	c.workqueue.Add(&types.Event{
		Type: types.EventAdd,
		Object: kube.NewEndpoint(obj.(*corev1.Endpoints)),
	})
}

func (c *endpointsController) onUpdate(prev, curr interface{}) {
	prevEp := prev.(*corev1.Endpoints)
	currEp := curr.(*corev1.Endpoints)

	// 如果旧 obj 的版本比新的 obj 新, 那么就直接跳过.
	if prevEp.GetResourceVersion() >= currEp.GetResourceVersion() {
		return
	}

	...

	// 向 workqueue 插入新增事件
	c.workqueue.Add(&types.Event{
		Type: types.EventUpdate,
		Object: kube.NewEndpoint(currEp),
	})
}

func (c *endpointsController) onDelete(obj interface{}) {
	ep, ok := obj.(*corev1.Endpoints)
	if !ok {
		// 如果类型不对, 则尝试转换为标准的 corev1.Endpoints
		ep = tombstone.Obj.(*corev1.Endpoints)
	}

	...

	// 向 workqueue 插入删除事件
	c.workqueue.Add(&types.Event{
		Type:   types.EventDelete,
		Object: kube.NewEndpoint(ep),
	})
}
```

### 启动 endpointsController 控制器

```go
func (c *endpointsController) run(ctx context.Context) {
	// 等待 endpoints 资源同步完成, 本质是等待 list-watch 的 list 拉取完成.
	if ok := cache.WaitForCacheSync(ctx.Done(), c.epInformer.HasSynced); !ok {
		return
	}

	handler := func() {
		for {
			// 从 workqueue 获取 obj
			obj, shutdown := c.workqueue.Get()
			if shutdown {
				return
			}

			// 调用同步程序来同步配置
			err := c.sync(ctx, obj.(*types.Event))

			// 标记该 obj 已完成
			c.workqueue.Done(obj)
		}
	}

	// 启动一个协程去执行 handler
	for i := 0; i < c.workers; i++ {
		go handler()
	}

	<-ctx.Done()
}
```

### 核心的 sync 同步配置逻辑

判断 ep 是否在 lister 里面存在, 不存在使用 `syncEmptyEndpoint` 进行同步, 反之则调用 `syncEndpoint` 进行配置同步.

```go
func (c *endpointsController) sync(ctx context.Context, ev *types.Event) error {
	ep := ev.Object.(kube.Endpoint)

	ns, err := ep.Namespace()
	if err != nil {
		return err
	}

	// 从 eplister 里获取 ns 和 serviceName 的 ep 结构
	newestEp, err := c.epLister.GetEndpoint(ns, ep.ServiceName())
	if err != nil {
		// 如果不存在该 ep, 则同步一个空的 endpoints
		if errors.IsNotFound(err) {
			return c.syncEmptyEndpoint(ctx, ep)
		}
		return err
	}

	// 同步 endpoints 配置
	return c.syncEndpoint(ctx, newestEp)
}
```

`syncEndpoint` 的主要逻辑通过遍历 spec.ports 和 subsets 构建 UpstreamNodes 数据结构, 然后调用 `SyncUpstreamNodesChangeToCluster` 同步配置.

```go
func (c *baseEndpointController) syncEndpoint(ctx context.Context, ep kube.Endpoint) error {
	// 获取 ep 的 ns
	namespace, err := ep.Namespace()
	if err != nil {
		return err
	}

	// 获取 ep 的 serviceName
	svcName := ep.ServiceName()

	// 通过 service lister 获取 ns 和 name 对应的 service 资源对象
	svc, err := c.svcLister.Services(namespace).Get(svcName)
	if err != nil {
		// 为空则推送空的 ep 结构
		if k8serrors.IsNotFound(err) {
			return c.syncEmptyEndpoint(ctx, ep)
		}
		return err
	}

	switch c.Kubernetes.APIVersion {
	case config.ApisixV2beta3:
		...
		// beta 就不分析了.

	case config.ApisixV2:
		// 获取 subsets, 这里的subsets 可以理解为服务对应的标签, 比如 v1, v2.
		var subsets []configv2.ApisixUpstreamSubset
		subsets = append(subsets, configv2.ApisixUpstreamSubset{})
		auKube, err := c.apisixUpstreamLister.V2(namespace, svcName)
		if err != nil {
			if !k8serrors.IsNotFound(err) {
				return err
			}
		} else if auKube.V2().Spec != nil && len(auKube.V2().Spec.Subsets) > 0 {
			subsets = append(subsets, auKube.V2().Spec.Subsets...)
		}

		// 从 apisix 接口拿到所有的 cluster 对象
		clusters := c.APISIX.ListClusters()

		for _, port := range svc.Spec.Ports {
			for _, subset := range subsets {
				// 把 endpoints 和 labels 选择标签转换成 UpstreamNodes 数据结构 
				nodes, err := c.translator.TranslateEndpoint(ep, port.Port, subset.Labels)
				...
				// 获取格式化的 name, 其实就是由这些参数拼凑了一个 name.
				name := apisixv1.ComposeUpstreamName(namespace, svcName, subset.Name, port.Port, types.ResolveGranularity.Endpoint)

				// 同步配置
				for _, cluster := range clusters {
					if err := c.SyncUpstreamNodesChangeToCluster(ctx, cluster, nodes, name); err != nil {
						return err
					}
				}
			}
		}
	default:
		panic(fmt.Errorf("unsupported ApisixUpstream version %v", c.Kubernetes.APIVersion))
	}
	return nil
}
```

从 apisix 中获取当前已有的 `upstream` 对象, 然后赋值新的 nodes 结构, 接着调用 `SyncManifests` 同步配置.

```go
func (c *Common) SyncUpstreamNodesChangeToCluster(ctx context.Context, cluster apisix.Cluster, nodes apisixv1.UpstreamNodes, upsName string) error {
	// 从 apisix 中获取该 name 的 upstream 对象.
	upstream, err := cluster.Upstream().Get(ctx, upsName)
	if err != nil {
		if err == apisixcache.ErrNotFound {
			return nil
		} else {
			return err
		}
	}

	// 更新 upstream 的 nodes 主机列表字段.
	upstream.Nodes = nodes

	updated := &utils.Manifest{
		Upstreams: []*apisixv1.Upstream{upstream},
	}

	// 调用 SyncManifests 只处理 updated 更新的数据.
	return c.SyncManifests(ctx, nil, updated, nil)
}
```

### 操作 apisix upstream 接口

`SyncManifests` 方法是 apisix ingress 里同步所有配置的关键入口. 其内部会依次判断各个对象列表, 当不为空时, 则调用对应的方法执行同步.

代码位置: `pkg/providers/utils/manifest.go`

```go
func SyncManifests(ctx context.Context, apisix apisix.APISIX, clusterName string, added, updated, deleted *Manifest) error {
	var merr *multierror.Error

	// 只处理新增数据
	if added != nil {
		...
	}
	// 只处理需要更新的数据
	if updated != nil {
		// 遍历更新 upstream 结构.
		for _, r := range updated.Upstreams {
			// 调用 apisix 执行 upstream 更新操作
			if _, err := apisix.Cluster(clusterName).Upstream().Update(ctx, r); err != nil {
				merr = multierror.Append(merr, err)
			}
		}
		...
	}
	// 只处理删除的数据
	if deleted != nil {
		...
	}
	if merr != nil {
		return merr
	}
	return nil
}
```

代码位置: `pkg/apisix/upstream.go`

```go
func (u *upstreamClient) Update(ctx context.Context, obj *v1.Upstream) (*v1.Upstream, error) {
	// 通过 json 序列化结构体 
	body, err := json.Marshal(obj)
	if err != nil {
		return nil, err
	}

	// 拼凑 apisix api 地址, 最后格式是这样. /apisix/admin/upstreams/{id}
	url := u.url + "/" + obj.ID

	// updateResource 封装了 http client put 更新操作
	resp, err := u.cluster.updateResource(ctx, url, "upstream", body)
	if err != nil {
		return nil, err
	}

	ups, err := resp.upstream()
	if err != nil {
		return nil, err
	}

	// 把返回的结构体插入到通过 `go-memdb` 实现的缓存中.
	if err := u.cluster.cache.InsertUpstream(ups); err != nil {
		return nil, err
	}
	return ups, err
}
```

## ingress gateway 实现原理

apisix ingress controller 也是支持 k8s gateway 资源处理. 默认不启动, 限于篇幅就不做分析. 

## apisix ingress 多级索引缓存设计

apisix ingress 内部使用 `go-memdb` 来构建多索引的缓存. `go-memdb` 是 hashicorp 社区的一个项目, 该库实现了实现内存级数据库, 支持丰富的索引, 支持多表的事务和多版本控制 mvcc 等.

**go-memdb 的项目地址:**

[https://github.com/hashicorp/go-memdb](https://github.com/hashicorp/go-memdb)

### 为什么使用 `go-memdb` 实现缓存系统 ? 

因为 `go-memdb` 作为数据库是支持索引的, 且索引类别很是丰富, 不仅单字段索引, 类似 mysql 的联合索引, 另外如果索引字段为数字, 还可以 range 范围查询.

可以想象如果不使用 go-memdb, 而使用自定义索引映射, 那会相当的麻烦. 比如你的 struct 有 3 个字段, 后面想通过这三个字段的值直接找到对应的对象, 当然不能粗暴遍历, 通常需要多个 map[struct]interface{} 自定义索引关系, 插入还好, 更麻烦的是当触发更新和删除时, 需要维护已建立的索引.

一句话, 维护索引关系显得相当麻烦.

### cache 实现原理

apisix ingress cache 的 schema 的代码位置.

在 db 里实例化了多个表结构, 每个表里又实例化多个索引映射. 当对 db 进行读写删操作时, `go-memdb` 会自动创建索引.

代码位置: `pkg/apisix/cache/schema.go`

```go
_schema = &memdb.DBSchema{
	Tables: map[string]*memdb.TableSchema{
		"route": {
			Name: "route",
			Indexes: map[string]*memdb.IndexSchema{
				"id": {
					Name:    "id",
					Unique:  true,
					Indexer: &memdb.StringFieldIndex{Field: "ID"},
				},
				"name": {
					Name:         "name",
					Unique:       true,
					Indexer:      &memdb.StringFieldIndex{Field: "Name"},
					AllowMissing: true,
				},
				"upstream_id": {
					Name:         "upstream_id",
					Unique:       false,
					Indexer:      &memdb.StringFieldIndex{Field: "UpstreamId"},
					AllowMissing: true,
				},
				...
			},
		},
		"upstream": {
			Name: "upstream",
			Indexes: map[string]*memdb.IndexSchema{
				"id": {
					Name:    "id",
					Unique:  true,
					Indexer: &memdb.StringFieldIndex{Field: "ID"},
				},
				"name": {
					Name:         "name",
					Unique:       true,
					Indexer:      &memdb.StringFieldIndex{Field: "Name"},
					AllowMissing: true,
				},
			},
		},
		...
	}
}
```

`dbcache` 里面实现了很多 apisix 内置对象的缓存管理, 下面拿 upstream 结构的缓存举例说明.

源码位置: `pkg/apisix/cache/memdb.go`

```go
type dbCache struct {
	db *memdb.MemDB
}

func NewMemDBCache() (Cache, error) {
	db, err := memdb.NewMemDB(_schema)
	if err != nil {
		return nil, err
	}
	return &dbCache{
		db: db,
	}, nil
}

// 往 upstream 表里写对象
func (c *dbCache) InsertUpstream(u *v1.Upstream) error {
	return c.insert("upstream", u.DeepCopy())
}

// 调用 memdb 的 insert 写数据
func (c *dbCache) insert(table string, obj interface{}) error {
	txn := c.db.Txn(true)
	defer txn.Abort()
	if err := txn.Insert(table, obj); err != nil {
		return err
	}
	txn.Commit()
	return nil
}

// 从 upstream 表里获取数据
func (c *dbCache) GetUpstream(id string) (*v1.Upstream, error) {
	obj, err := c.get("upstream", id)
	if err != nil {
		return nil, err
	}
	return obj.(*v1.Upstream).DeepCopy(), nil
}

// tnx.first 是获取第一条数据
func (c *dbCache) get(table, id string) (interface{}, error) {
	txn := c.db.Txn(false)
	defer txn.Abort()
	obj, err := txn.First(table, "id", id)
	if err != nil {
		if err == memdb.ErrNotFound {
			return nil, ErrNotFound
		}
		return nil, err
	}
	if obj == nil {
		return nil, ErrNotFound
	}
	return obj, nil
}

// txn.get 可以获取多条数据
func (c *dbCache) list(table string) ([]interface{}, error) {
	txn := c.db.Txn(false)
	defer txn.Abort()
	iter, err := txn.Get(table, "id")
	if err != nil {
		return nil, err
	}
	var objs []interface{}
	for obj := iter.Next(); obj != nil; obj = iter.Next() {
		objs = append(objs, obj)
	}
	return objs, nil
}
...
```