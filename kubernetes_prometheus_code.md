# 源码分析 kubernetes prometheus 监控系统服务发现实现原理

> 本文基于 prometheus v2.41.0 版本进行源码分析.

下图为 prometheus k8s 服务发现的流程原理.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301291712406.png)

下图为 prometheus k8s provider 的流程原理.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301291806590.png)

## prometheus provider 初始化入口

`ApplyConfig` 用来实例化 service discovery provider 对象, 并把 sd provider 注册到集合里, 然后遍历集合启动 provider.

代码位置: `discovery/manager.go`

```go
func (m *Manager) ApplyConfig(cfg map[string]Configs) error {
	m.mtx.Lock()
	defer m.mtx.Unlock()

	// 创建 service discovery provider 对象, 并进行注册到 Manager 里.
	var failedCount int
	for name, scfg := range cfg {
		failedCount += m.registerProviders(scfg, name)
	}

	var (
		wg sync.WaitGroup
		newProviders []*Provider
	)

	// 遍历已注册的 providers 服务发现对象
	for _, prov := range m.providers {
		...

		newProviders = append(newProviders, prov)
		// 如果没启动则进行启动.
		if !prov.IsStarted() {
			m.startProvider(m.ctx, prov)
		}

		...
	}

	// 赋值
	m.providers = newProviders
	return nil
}

// 根据配置构建并注册 provider.
func (m *Manager) registerProviders(cfgs Configs, setName string) int {
	add := func(cfg Config) {
		// 构建 discoverer 发现者对象
		d, err := cfg.NewDiscoverer(DiscovererOptions{
		})
		...
		// 添加到 providers 集合里
		m.providers = append(m.providers, &Provider{
			name:   fmt.Sprintf("%s/%d", typ, m.lastProvider),
			d:      d,
			config: cfg,
		})
		added = true
	}
	return failed
}
```

`startProvider` 启动 sd provider, 内部会启动两个协程, 一个是启动 provider 进行发现, 另一个是启动更新监听.

```go
func (m *Manager) startProvider(ctx context.Context, p *Provider) {
	ctx, cancel := context.WithCancel(ctx)
	updates := make(chan []*targetgroup.Group)

	p.cancel = cancel

	// 启动 service discovery 服务发现
	go p.d.Run(ctx, updates)

	// 启动更新监听逻辑
	go m.updater(ctx, p, updates)
}
```

## k8s prometheus 服务发现的实现

prometheus 支持 k8s 下面维度的服务发现.

```go
// Role is role of the service in Kubernetes.
type Role string

// The valid options for Role.
const (
	RoleNode          Role = "node"
	RolePod           Role = "pod"
	RoleService       Role = "service"
	RoleEndpoint      Role = "endpoints"
	RoleEndpointSlice Role = "endpointslice"
	RoleIngress       Role = "ingress"
)
```

根据不同的 role 来创建不同的 discoverer, 为每个 namespace 创建独立的 discoverer, 后面启动这些实例化的 discoverer.

源码位置: `discovery/kubernetes/kubernetes.go`

```go
func (d *Discovery) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
	d.Lock()
	namespaces := d.getNamespaces()

	switch d.role {
	case RoleEndpoint:
		for _, namespace := range namespaces {
			...
			// 创建 endpoints discoverer
			eps := NewEndpoints(
				d.newEndpointsByNodeInformer(elw),
				cache.NewSharedInformer(slw, &apiv1.Service{}, resyncDisabled),
				cache.NewSharedInformer(plw, &apiv1.Pod{}, resyncDisabled),
				nodeInf,
			)
			d.discoverers = append(d.discoverers, eps)

			// 启动 informer
			go eps.endpointsInf.Run(ctx.Done())
			go eps.serviceInf.Run(ctx.Done())
			go eps.podInf.Run(ctx.Done())
		}
	case RolePod:
		var nodeInformer cache.SharedInformer
		if d.attachMetadata.Node {
			nodeInformer = d.newNodeInformer(ctx)
			go nodeInformer.Run(ctx.Done())
		}

		// 为每个 namespace 创建 pod discoverer
		for _, namespace := range namespaces {
			p := d.client.CoreV1().Pods(namespace)
			plw := &cache.ListWatch{
				ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
					// 只匹配设定的 labels 标签
					options.FieldSelector = d.selectors.endpoints.field
					options.LabelSelector = d.selectors.endpoints.label
					return e.List(ctx, options)
				},
				WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
					// 只匹配设定的 labels 标签
					options.FieldSelector = d.selectors.endpoints.field
					options.LabelSelector = d.selectors.endpoints.label
					return e.List(ctx, options)
				},
			}

			// 创建 pod discoverer
			pod := NewPod(
				log.With(d.logger, "role", "pod"),
				d.newPodsByNodeInformer(plw),
				nodeInformer,
			)

			d.discoverers = append(d.discoverers, pod)
			// 启动 pod informer
			go pod.podInf.Run(ctx.Done())
		}
	case RoleService:
		...
	case RoleIngress:
		for _, namespace := range namespaces {
			ingress := NewIngress(
				informer,
			)
			d.discoverers = append(d.discoverers, ingress)
			go ingress.informer.Run(ctx.Done())
		}
	case RoleNode:
		nodeInformer := d.newNodeInformer(ctx)
		node := NewNode(log.With(d.logger, "role", "node"), nodeInformer)
		d.discoverers = append(d.discoverers, node)
		go node.informer.Run(ctx.Done())
	default:
	}

	var wg sync.WaitGroup
	// 启动不同的 discoverer
	for _, dd := range d.discoverers {
		wg.Add(1)
		go func(d discovery.Discoverer) {
			defer wg.Done()
			d.Run(ctx, ch)
		}(dd)
	}

	d.Unlock()

	wg.Wait()
	<-ctx.Done()
}
```

k8s 支持的 role 有很多种, 这里就分析 pod 和 node role, 它们的实现原理大同小异.

### 自动发现 k8s 里 pod 资源信息

代码位置: `discovery/kubernetes/pod.go`

#### 初始化 pod discovery

```go
// NewPod creates a new pod discovery.
func NewPod(l log.Logger, pods cache.SharedIndexInformer, nodes cache.SharedInformer) *Pod {
	// 构建 pod 对象
	p := &Pod{
		podInf:           pods,
		nodeInf:          nodes,
		withNodeMetadata: nodes != nil,
		store:            pods.GetStore(),
		logger:           l,
		queue:            workqueue.NewNamed("pod"),
	}

	// 在 node informer 里注册自定义 eventHandler
	_, err := p.podInf.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(o interface{}) {
			p.enqueue(o)
		},
		DeleteFunc: func(o interface{}) {
			p.enqueue(o)
		},
		UpdateFunc: func(_, o interface{}) {
			// 把更新的 pod 对象推入队列中, 旧对象忽略.
			p.enqueue(o)
		},
	})
	...

	if p.withNodeMetadata {
		_, err = p.nodeInf.AddEventHandler(cache.ResourceEventHandlerFuncs{
			AddFunc: func(o interface{}) {
				node := o.(*apiv1.Node)
				p.enqueuePodsForNode(node.Name)
			},
			UpdateFunc: func(_, o interface{}) {
				node := o.(*apiv1.Node)
				p.enqueuePodsForNode(node.Name)
			},
			DeleteFunc: func(o interface{}) {
				node := o.(*apiv1.Node)
				p.enqueuePodsForNode(node.Name)
			},
		})
	}

	return p
}

// 把 pod 的格式化名字推入到 workqueue 队列中.
func (p *Pod) enqueue(obj interface{}) {
	key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
	if err != nil {
		return
	}

	p.queue.Add(key)
}

// 通过 informer lister 获取 name 对应的 pods, 并遍历推入到 workqueue 队列中.
func (p *Pod) enqueuePodsForNode(nodeName string) {
	// 通过 nodename 获取 pods 集合.
	pods, err := p.podInf.GetIndexer().ByIndex(nodeIndex, nodeName)
	if err != nil {
		return
	}

	for _, pod := range pods {
		p.enqueue(pod.(*apiv1.Pod))
	}
}
```

#### 启动 pod discovery

```go
func (p *Pod) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
	defer p.queue.ShutDown()

	cacheSyncs := []cache.InformerSynced{p.podInf.HasSynced}
	if p.withNodeMetadata {
		cacheSyncs = append(cacheSyncs, p.nodeInf.HasSynced)
	}

	// 等待所有的 informer 把数据同步到本地缓存.
	if !cache.WaitForCacheSync(ctx.Done(), cacheSyncs...) {
		return
	}

	go func() {
		// 异步循环调用 process.
		for p.process(ctx, ch) {
		}
	}()

	// 等待上层关闭.
	<-ctx.Done()
}

func (p *Pod) process(ctx context.Context, ch chan<- []*targetgroup.Group) bool {
	keyObj, quit := p.queue.Get()
	if quit {
		return false
	}
	defer p.queue.Done(keyObj)
	key := keyObj.(string)

	// 通过 key 获取 namespace 和 pod name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return true
	}

	// 从 cache storge 获取是否对象, 没有则说明是触发了 delete event 事件
	o, exists, err := p.store.GetByKey(key)
	if err != nil {
		return true
	}
	// 不存在
	if !exists {
		// 通知通信, 这里其实是删除更新.
		send(ctx, ch, &targetgroup.Group{Source: podSourceFromNamespaceAndName(namespace, name)})
		return true
	}

	// interface to pod
	pod, err := convertToPod(o)
	if err != nil {
		return true
	}

	// 构建 targetgroup 对象, 内有 podip, podname 等字段信息.
	send(ctx, ch, p.buildPod(pod))
	return true
}
```

通过 k8s pod 对象组装 targetgroup 对象, 打入 labels 标签信息.

```go
func (p *Pod) buildPod(pod *apiv1.Pod) *targetgroup.Group {
	tg := &targetgroup.Group{
		// 格式化 pod 名字 "pod/" + namespace + "/" + name
		Source: podSource(pod),
	}

	// 如果 podIP 为空, 则没必要抓取
	if len(pod.Status.PodIP) == 0 {
		return tg
	}

	// 打入标签
	tg.Labels = podLabels(pod)
	tg.Labels[namespaceLabel] = lv(pod.Namespace)

	containers := append(pod.Spec.Containers, pod.Spec.InitContainers...)
	for i, c := range containers {
		...
		cID := p.findPodContainerID(cStatuses, c.Name)

		for _, port := range c.Ports {
			ports := strconv.FormatUint(uint64(port.ContainerPort), 10)
			// addr 的格式为 ip:port
			addr := net.JoinHostPort(pod.Status.PodIP, ports)

			// targetgroup 里每个 container 都是一个 target 对象.
			// 另外不同的端口也是一个 target.
			tg.Targets = append(tg.Targets, model.LabelSet{
				model.AddressLabel:            lv(addr),
				podContainerNameLabel:         lv(c.Name),
				podContainerIDLabel:           lv(cID),
				podContainerImageLabel:        lv(c.Image),
				podContainerPortNumberLabel:   lv(ports),
				podContainerPortNameLabel:     lv(port.Name),
				podContainerPortProtocolLabel: lv(string(port.Protocol)),
				podContainerIsInit:            lv(strconv.FormatBool(isInit)),
			})
		}
	}

	return tg
}
```

send 方法是把 `targetgroup` 对象推到 ch 里, 这个 `ch` 由 `manager.updater` 来消费处理.

```go
func send(ctx context.Context, ch chan<- []*targetgroup.Group, tg *targetgroup.Group) {
	if tg == nil {
		return
	}
	select {
	case <-ctx.Done():
	case ch <- []*targetgroup.Group{tg}:
	}
}
```

### 自动发现 k8s 里 node 资源信息

k8s node 的服务发现跟 k8s pod 类似, pod 服务发现是把 pod 对象封装成 targetoup 进行收集, 而 node 服务发现则是把 node 信息作为 targetgroup 进行收集, 这里的 node 其实就是 kubelet.

源码位置: `discovery/kubernetes/node.go`

> 篇幅原因这里省略 node informer 注册 eventHandler 的代码...

```go
func (n *Node) process(ctx context.Context, ch chan<- []*targetgroup.Group) bool {
	keyObj, quit := n.queue.Get()
	if quit {
		return false
	}
	defer n.queue.Done(keyObj)
	key := keyObj.(string)

	_, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return true
	}

	o, exists, err := n.store.GetByKey(key)
	if err != nil {
		return true
	}
	if !exists {
		send(ctx, ch, &targetgroup.Group{Source: nodeSourceFromName(name)})
		return true
	}
	node, err := convertToNode(o)
	if err != nil {
		return true
	}
	send(ctx, ch, n.buildNode(node))
	return true
}

func (n *Node) buildNode(node *apiv1.Node) *targetgroup.Group {
	tg := &targetgroup.Group{
		Source: nodeSource(node),
	}
	// 赋值 targetgroup labels
	tg.Labels = nodeLabels(node)

	// 从 node 对象里获取 addr, 拿不到则退出.
	addr, addrMap, err := nodeAddress(node)
	if err != nil {
		return nil
	}

	// 组装 addr 格式为 ip:port
	addr = net.JoinHostPort(addr, strconv.FormatInt(int64(node.Status.DaemonEndpoints.KubeletEndpoint.Port), 10))

	// labels 里加入 addr 和 node 名字.
	t := model.LabelSet{
		model.AddressLabel:  lv(addr),
		model.InstanceLabel: lv(node.Name),
	}

	for ty, a := range addrMap {
		ln := strutil.SanitizeLabelName(nodeAddressPrefix + string(ty))
		t[model.LabelName(ln)] = lv(a[0])
	}
	// 赋值到 target 抓取对象列表里.
	tg.Targets = append(tg.Targets, t)

	return tg
}
```

`nodeAddress` 方法会从 node 的 `Status.Addresses` 里获取可以用来作为 target 的地址. 通过代码可以分析出 node 地址的优先级.

1. NodeInternalIP
2. NodeInternalDNS
3. NodeExternalIP
4. NodeExternalDNS
5. NodeLegacyHostIP
6. NodeHostName

```go
// nodeAddresses returns the provided node's address, based on the priority:
func nodeAddress(node *apiv1.Node) (string, map[apiv1.NodeAddressType][]string, error) {
	m := map[apiv1.NodeAddressType][]string{}
	for _, a := range node.Status.Addresses {
		m[a.Type] = append(m[a.Type], a.Address)
	}

	if addresses, ok := m[apiv1.NodeInternalIP]; ok {
		return addresses[0], m, nil
	}
	if addresses, ok := m[apiv1.NodeInternalDNS]; ok {
		return addresses[0], m, nil
	}
	if addresses, ok := m[apiv1.NodeExternalIP]; ok {
		return addresses[0], m, nil
	}
	if addresses, ok := m[apiv1.NodeExternalDNS]; ok {
		return addresses[0], m, nil
	}
	if addresses, ok := m[apiv1.NodeAddressType(NodeLegacyHostIP)]; ok {
		return addresses[0], m, nil
	}
	if addresses, ok := m[apiv1.NodeHostName]; ok {
		return addresses[0], m, nil
	}
	return "", m, errors.New("host address unknown")
}
```

## discovery manager 处理动态更新信号

`manager.updater` 收到 sd provider 发来的通知, 先调用 `updateGroup` 方法更新 target 目标集合, 再发个通知给 `triggerSend`.

代码位置: `discovery/manager.go`

```go
func (m *Manager) updater(ctx context.Context, p *Provider, updates chan []*targetgroup.Group) {
	defer m.cleaner(p)
	for {
		select {
		case <-ctx.Done():
			return
		case tgs, ok := <-updates:
			if !ok {
				<-ctx.Done()
				return
			}

			p.mu.RLock()
			for s := range p.subs {
				// 把新增的 targetgroup 更新到 manager 的 target 集合里.
				m.updateGroup(poolKey{setName: s, provider: p.name}, tgs)
			}
			p.mu.RUnlock()

			select {
			case m.triggerSend <- struct{}{}:
				// 给 triggersend 管道通知
			default:
			}
		}
	}
}
```

`manager.sender` 每隔 5s 会监听消费由 `updater` 通知的 `triggerSend` 管道, 然后通知给 `syncCh` 管道.

```go
func (m *Manager) sender() {
	// 创建一个 5 秒的定时器
	ticker := time.NewTicker(m.updatert)
	defer ticker.Stop()

	for {
		select {
		case <-m.ctx.Done():
			return
		case <-ticker.C:
			select {
			case <-m.triggerSend:
				select {
				case m.syncCh <- m.allGroups():
				default:
					// 如果 channel 满了, 那么再次发个更新通知.
					select {
					case m.triggerSend <- struct{}{}:
					default:
					}
				}
			default:
			}
		}
	}
}
```

## scrape manager 抓取管理的实现

`scrapeManager` 是 prometheus 用来抓取 targets metrics 指标的管理者. 

### scrape manager 入口

下面是 scrape manager 的启动入口, 在 scrapeManager 启动时会传入一个 discoveryManager 的 syncCh, 该 syncCh 会由 discovery manager 进行通知同步信号的.

源码位置: `cmd/prometheus/main.go`

```go
// Scrape manager.
g.Add(
	func() error {
		<-reloadReady.C

		// scrape manager 启动方法
		err := scrapeManager.Run(discoveryManagerScrape.SyncCh())
		return err
	},

	func(err error) {
		// scrape manager 关闭方法
		scrapeManager.Stop()
	},
)
```

### scrape manager reload 设计

源码位置: `scrape/manager.go`

```go
func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
	go m.reloader()
	for {
		select {
		case ts := <-tsets:
			// 更新 scrape pool 
			m.updateTsets(ts)

			select {
			case m.triggerReload <- struct{}{}:
				// 通知 reloader 进行 reload.
			default:
			}

		case <-m.graceShut:
			// 退出
			return nil
		}
	}
}
```

`reloader` 每隔 `5s` 检查是否有 triggerReload 信号通知, 当有信号时调用 `reload()` 进行配置更新.

```go
func (m *Manager) reloader() {
	ticker := time.NewTicker(time.Duration(reloadIntervalDuration))
	defer ticker.Stop()

	for {
		select {
		case <-m.graceShut:
			return
		case <-ticker.C:
			select {
			case <-m.triggerReload:
				m.reload()
			case <-m.graceShut:
				return
			}
		}
	}
}

func (m *Manager) reload() {
	m.mtxScrape.Lock()
	var wg sync.WaitGroup
	for setName, groups := range m.targetSets {
		// 当没有 setname 时, 则进行创建 scrapepool 抓取池
		if _, ok := m.scrapePools[setName]; !ok {
			scrapeConfig, ok := m.scrapeConfigs[setName]
			if !ok {
				continue
			}
			// 实例化新的 scrape pool 抓取池
			sp, err := newScrapePool(scrapeConfig, m.append, m.jitterSeed, log.With(m.logger, "scrape_pool", setName), m.opts)
			if err != nil {
				continue
			}
			m.scrapePools[setName] = sp
		}

		wg.Add(1)
		go func(sp *scrapePool, groups []*targetgroup.Group) {
			// 同步配置到 sp 的 activeTargets 目标集合里.
			sp.Sync(groups)
			wg.Done()
		}(m.scrapePools[setName], groups)

	}
	m.mtxScrape.Unlock()
	wg.Wait()
}
```

简单说 scrape pool 实现了一组 target 的抓取和管理, scrape manager 可以对 scrape pool 的 target 进行配置同步管理, 也可以启动 sp 使其开始抓取 target 数据.

本文主要分析 prometheus k8s 的服务发现, scrape pool 不是重点, 其实现细节就不再描述了, 

## 结论

本文主要介绍了 prometheus 如何实现 k8s 的服务发现, 主要分析了 pod 和 node 资源对象的服务发现.

其原理还是很简单，通过 k8s 的 apiserver 获取 k8s 里资源对象的信息, 比如 pod, node, ingress 等. 获取到资源对象构建成 targetGroup 目标组结构, 目标组内含有目标的地址信息及其他后期便于检索的 labels 标签信息. 接着把 targetGroup 信息通知给 scape manager.

`scapeManager` 收到更新通知后, 根据配置变动决定是动态更新 targets 还是新增 scrapePool 池对象. 如果是新增 targetGroup 不仅需要创建, 而且还需启动 scrapePool 池.

另外 prometheus k8s 内部的配置通知的逻辑实现有点绕, 下图整理了配置监听, 通知传递和动态配置更新的过程.

1. 首先 k8s provider 的 node 和 pod discoverer 监听到配置变动后, 传递给由 `manager.updater` 消费的管道.
2. discovery updater 把监听到的信号再通知给内部的 `triggerSend` 管道.
3. discovery sender 把从 `triggerSend` 收到的信号再传递给 `syncCh`.
4. scrapeManager 初始化阶段会启动 `Run()` 方法监听 `syncCh` 管道, 把收到的信号再传递给 `triggerReload` 管道.
5. scapeManager 的 reloader 协程会监听 `triggerReload` 信号, 执行最后的配置同步更新操作.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301291806590.png)