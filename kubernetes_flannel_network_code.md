# 源码分析 kubernetes CNI flannel 容器网络插件的设计实现原理

> 本文基于 flannel `v0.21.0` 版本源码分析.

CNI, 它的全称是 `Container Network Interface`, 即容器网络的 API 接口. 平时比较常用的 CNI 实现有 Flannel、Calico 等.

Flannel 支持 overlay/underlay 两种网络模式. 它使用 etcd 或者 kubernetes apiserver 存储整个集群的网络配置. 每个kubernetes 节点上会运行 flanneld 服务组件, 它从 etcd 或者 kubernetes api 中获取集群中各个 node 的网络地址 subset, 然后进行网络路由配置, 这样就可以实现跨主机的容器之间网络通信. 

flannel 目前支持 udp, vxlan, host-gw 等 backend 实现. 

- `udp` 模式少有人用, 其实现原理是在用户态实现 udp 转发服务, 数据会在内核和用户态之间拷贝, 从而影响转发性能. 
- `host-gw` (underlay 模式)  通过三层路由的方式实现通信, 不涉及vxlan 这类的封包解包, 所以也不需要 flannel.1 虚机网卡, 直接配置路由表的方式设置 pod 的下一跳, 达到实现跨主机的容器之间的通信的目的. flannel host-gw 方案无疑是性能最好的方案, 但需要 node 之间同在一个二层网络 (vlan) 里可达, 如果 node 之间不在同一个二层网络, 那么则需要使用 `calico` 这类路由网络方案.
- `vxlan` (overlay 模式) 是 Flannel 默认和推荐的模式, vxlan 网络虚拟化技术, 它使用一种隧道协议, 将二层以太网帧封装在四层 UDP 报文中, 通过三层网络传输组成一个虚拟大二层网络.

`host-gw` 的性能损失大约在 10% 左右，而 vxlan 这类网络方案性能损失在 20%~30% 左右.

本文主要介绍 vxlan 和 host-gw 的 backend 下的跨主机容器通信过程及原理.

**flannel 的设计实现基本流程**

下图 subnet manager 使用 k8s kube-apiserver, backend 选用 vxlan 网络.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302041931615.png)

下图 subnet manager 使用 k8s kube-apiserver, backend 选用 host-gw 网络.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302051020026.png)

## flannel main 启动入口

分析下 main 的运行流程原理.

1. 实例化 subnet manager 管理对象, 当启用 `kube-subnet-mgr` 参数时, 使用 k8s apiserver 作为 subsetmgr, 其他情况使用 etcd 作为 subsetmgr.
2. 根据 flannel backend 类型创建 backend 对象.
3. 根据 backend 里配置创建获取 network 控制器对象.
4. 启动 backend 的 network 控制器, 核心的处理逻辑都是各个 backend 的 network 里.
5. 如果使用 systemd 管理进程, 则用 uds 跟 systemd 建连, 发送就绪消息.
6. 等需要退出时, 向 apiserver 发送状态请求, 表明当前 flannel 在运行.
7. 等待所有协程退出.

这里涉及到了几个组件.

- subnet manager, 实现了ip的租期管理, 申请，续约, 监听事件变动都是在这里实现的.
- backend manager, 用来构建不同容器通信的 backend 对象. 启动阶段 udp, vxlan, host-gw 的 backend 都注册在这里, 通过 `GetBackend` 获取对应类型的 backend 对象.
- network, 在各个 backend 里都有实现 network 控制器组件, 该组件用来真正的去实施网络策略配置.

```go
func main() {
	// ...

	ctx, cancel := context.WithCancel(context.Background())

	// 实例化 subnet manager, 当启用 kube-subnet-mgr 参数时, 使用 k8s apiserver 作为 subsetmgr, 其他情况使用 etcd 作为 subsetmgr.
	sm, err := newSubnetManager(ctx)
	if err != nil {
		os.Exit(1)
	}
	log.Infof("Created subnet manager: %s", sm.Name())

	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, os.Interrupt, syscall.SIGTERM)

	wg := sync.WaitGroup{}
	wg.Add(1)
	go func() {
		// 监听 signal 信号, 收到后 cancel 退出.
		shutdownHandler(ctx, sigs, cancel)
		wg.Done()
	}()

	// 获取 flannel 配置.
	config, err := getConfig(ctx, sm)
	if err == errCanceled {
		wg.Wait()
		os.Exit(0)
	}

	...

	// 根据 flannel 类型创建 backend 对象.
	bm := backend.NewManager(ctx, sm, extIface)
	be, err := bm.GetBackend(config.BackendType)
	if err != nil {
		cancel()
		wg.Wait()
		os.Exit(1)
	}

	// 根据配置在 backend 里创建获取 network 对象.
	bn, err := be.RegisterNetwork(ctx, &wg, config)
	if err != nil {
		cancel()
		wg.Wait()
		os.Exit(1)
	}

	// ...

	// 启动 backend 的 network.
	wg.Add(1)
	go func() {
		bn.Run(ctx)
		wg.Done()
	}()

	// 如果使用 systemd 管理进程, 则用 uds 跟 systemd 建连, 发送就绪消息.
	// 跟 systemd-notify 二进制一样的效果.
	_, err = daemon.SdNotify(false, "READY=1")
	if err != nil {
		log.Errorf("Failed to notify systemd the message READY=1 %v", err)
	}

	// 等需要退出时, 向 apiserver 发送状态请求, 表明当前 flannel 在运行. 
	err = sm.CompleteLease(ctx, bn.Lease(), &wg)
	if err != nil {
		log.Errorf("CompleteLease execute error err: %v", err)
	}

	// 等待所有协程退出.
	wg.Wait()
	os.Exit(0)
}
```

## subnetManager 原理

在 flannel 内部实现了两种 subnetManager, 一种是 k8s kube-apiserver, 另一种是 etcd. 这里拿 kube-apiserver 举例, 两者实现上大同小异, 无大差异.

### 创建 subnetManager 子网管理器

`newKubeSubnetManager` 会创建 node 资源的 informer 监听对象, 并注册 list/watch 过滤方法, 注册 eventHandler 事件回调方法. 

代码位置: `pkg/subnet/kube/kube.go`

```go
func newKubeSubnetManager(ctx context.Context, c clientset.Interface, sc *subnet.Config, nodeName, prefix string, useMultiClusterCidr bool) (*kubeSubnetManager, error) {
	var ksm kubeSubnetManager

	// 构建 annotations 里 key 的名字, 跟 prefix 前缀组合后格式为 `{prefix}/key`
	ksm.annotations, err = newAnnotations(prefix)
	if err != nil {
		return nil, err
	}

	// ipv4 开关
	ksm.enableIPv4 = sc.EnableIPv4
	// ipv6 开关
	ksm.enableIPv6 = sc.EnableIPv6
	// kubeclient
	ksm.client = c
	ksm.nodeName = nodeName
	ksm.subnetConf = sc

	// events 用来实现事件通知, 事件由 informer 监听获取后推入的.
	ksm.events = make(chan subnet.Event, scale)

	// 如果类型为 alloc, 则无需启动 node informer 监听.
	if sc.BackendType == "alloc" {
		ksm.disableNodeInformer = true
	}
	if !ksm.disableNodeInformer {
		indexer, controller := cache.NewIndexerInformer(
			&cache.ListWatch{
				ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
					// 设定 list 过滤的条件
					return ksm.client.CoreV1().Nodes().List(ctx, options)
				},
				WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
					// 设定 watch 过滤的条件
					return ksm.client.CoreV1().Nodes().Watch(ctx, options)
				},
			},
			&v1.Node{}, // 监听 node 资源对象.
			resyncPeriod,
			cache.ResourceEventHandlerFuncs{
				AddFunc: func(obj interface{}) {
					// 注册 add event 回调方法
					ksm.handleAddLeaseEvent(subnet.EventAdded, obj)
				},
				// 注册 event 回调方法
				UpdateFunc: ksm.handleUpdateLeaseEvent,
				DeleteFunc: func(obj interface{}) {
					// 注册 delete event 回调方法
					_, isNode := obj.(*v1.Node)
					if !isNode {
						deletedState, ok := obj.(cache.DeletedFinalStateUnknown)
						if !ok {
							return
						}
						node, ok := deletedState.Obj.(*v1.Node)
						if !ok {
							return
						}
						obj = node
					}
					ksm.handleAddLeaseEvent(subnet.EventRemoved, obj)
				},
			},
			// informer indexer 索引方法.
			cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc},
		)
		ksm.nodeController = controller
		ksm.nodeStore = listers.NewNodeLister(indexer)
	}

	// 如果开启了多集群 cidr 支持, 则需要监听 k8s networkingv1alpha1.ClusterCIDR 资源.
	if useMultiClusterCidr {
		_, clusterController := cache.NewIndexerInformer(
			&cache.ListWatch{
				...
			},
			&networkingv1alpha1.ClusterCIDR{},
			resyncPeriod,
			cache.ResourceEventHandlerFuncs{
				AddFunc: func(obj interface{}) {
					// 注册 add 事件方法
					ksm.handleAddClusterCidr(obj)
				},
				DeleteFunc: func(obj interface{}) {
					// 注册 delete 事件方法
					ksm.handleDeleteClusterCidr(obj)
				},
			},
			cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc},
		)
		ksm.clusterCIDRController = clusterController
	}
	return &ksm, nil
}
```

`handleAddLeaseEvent` 是上面 informer 里 add/delete 的事件回调方法, 其逻辑根据 node 对象中的 spec.PodCIDR 和 annotations 一些字段来构建 lease 租约结构, 然后把 lease 写到管道里.

```go
func (ksm *kubeSubnetManager) handleAddLeaseEvent(et subnet.EventType, obj interface{}) {
	n := obj.(*v1.Node)
	// 如果 node 注释集合里没有 'kube-subnet-manager' 字段, 或者不为 true, 则直接跳出.
	if s, ok := n.Annotations[ksm.annotations.SubnetKubeManaged]; !ok || s != "true" {
		return
	}

	// 根据 node 对象中的 spec.PodCIDR 和 annotations 信息来构建 lease 租约结构.
	l, err := ksm.nodeToLease(*n)
	if err != nil {
		return
	}

	// 把解析到的 lease 租约结构写到 events 里做通知.
	ksm.events <- subnet.Event{Type: et, Lease: l}
}
```

`nodeToLease` 会根据 v1.Node 对象里的数据构建 subnet.Lease 数据结构. 其转换过程看下面代码中的注释.

```go
func (ksm *kubeSubnetManager) nodeToLease(n v1.Node) (l subnet.Lease, err error) {
	if ksm.enableIPv4 {
		// 从 node annotations 里获取 node 的 publicip
		l.Attrs.PublicIP, err = ip.ParseIP4(n.Annotations[ksm.annotations.BackendPublicIP])
		if err != nil {
			return l, err
		}
		// 获取 node annotations 的 BackendData 字段数据
		l.Attrs.BackendData = json.RawMessage(n.Annotations[ksm.annotations.BackendData])

		var cidr *net.IPNet
		switch {
		case len(n.Spec.PodCIDRs) == 0:
			// 无效的 cidr
			_, cidr, err = net.ParseCIDR(n.Spec.PodCIDR)
			if err != nil {
				return l, err
			}
		case len(n.Spec.PodCIDRs) < 3:
			// 如果 podCIDRS 小于 3 个, 循环遍历出一个 cidr 为 ipv4的.
			for _, podCidr := range n.Spec.PodCIDRs {
				_, parseCidr, err := net.ParseCIDR(podCidr)
				if err != nil {
					return l, err
				}
				if len(parseCidr.IP) == net.IPv4len {
					cidr = parseCidr
					break
				}
			}
		default:
			return l, fmt.Errorf("node %q pod cidrs should be IPv4/IPv6 only or dualstack", ksm.nodeName)
		}
		// 把 cidr 转成 IP4Net 格式, 记录 ip 和 size
		l.Subnet = ip.FromIPNet(cidr)
		// 当前 subnet 为 ipv4
		l.EnableIPv4 = ksm.enableIPv4
	}

	if ksm.enableIPv6 {
		// ipv6 跟 ipv4 差不多.
	}
	l.Attrs.BackendType = n.Annotations[ksm.annotations.BackendType]
	return l, nil
}
```

`WatchLeases` 用来监听 ksm.events 管道, 当有数据时返回给调用方.

```go
func (ksm *kubeSubnetManager) WatchLeases(ctx context.Context, cursor interface{}) (subnet.LeaseWatchResult, error) {
	select {
	case event := <-ksm.events:
		return subnet.LeaseWatchResult{
			Events: []subnet.Event{event},
		}, nil
	case <-ctx.Done():
		return subnet.LeaseWatchResult{}, context.Canceled
	}
}
```

## backend manager 实现原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302042054717.png)

`BackendManager` 根据 backendType 获取注册表里的工厂方法, 然后创建对应的 backend 实例对象. flannel 在启动阶段, 各个 backend 会注册工厂方法到 backendManager 里.

源码位置: `pkg/backend/manager.go`

```go
var constructors = make(map[string]BackendCtor)

type manager struct {
	ctx      context.Context
	sm       subnet.Manager
	mux      sync.Mutex
	active   map[string]Backend
	wg       sync.WaitGroup
}

func NewManager(ctx context.Context, sm subnet.Manager, extIface *ExternalInterface) Manager {
	return &manager{
		ctx:      ctx,
		sm:       sm,
		extIface: extIface,
		active:   make(map[string]Backend),
	}
}

func (bm *manager) GetBackend(backendType string) (Backend, error) {
	bm.mux.Lock()
	defer bm.mux.Unlock()

	betype := strings.ToLower(backendType)
	// 如果已存在, 则直接返回.
	if be, ok := bm.active[betype]; ok {
		return be, nil
	}

	// 在 constructors 里获取传入类型的 backend 工厂方法.
	befunc, ok := constructors[betype]
	if !ok {
		return nil, fmt.Errorf("unknown backend type: %v", betype)
	}

	// 创建 backend 对应.
	be, err := befunc(bm.sm, bm.extIface)
	if err != nil {
		return nil, err
	}

	// 注册到活动 backend 字典里.
	bm.active[betype] = be

	bm.wg.Add(1)
	go func() {
		<-bm.ctx.Done()

		// 退出时需要删除
		bm.mux.Lock()
		delete(bm.active, betype)
		bm.mux.Unlock()

		bm.wg.Done()
	}()

	return be, nil
}

func Register(name string, ctor BackendCtor) {
	// 注册 backend
	constructors[name] = ctor
}
```

看下 vxlan 和 host-gw 是如何注册进来的.

```go
// 源码位置: pkg/backend/vxlan/vxlan.go
func init() {
	backend.Register("vxlan", New)
}

// 源码位置: pkg/backend/hostgw/hostgw.go
func init() {
	backend.Register("host-gw", New)
}
```

## vxlan 网络通信的实现原理

下图为 vxlan 跨主机的网络通信架构.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302042019492.png)

下图为 vxlan network 的实现原理.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302042036994.png)

vxlan 的 network 会从 kubeSubnetManager informer 中获取全量及增量的 node 事件. 然后从调用 `handleSubnetEvents` 方法对 vxlan 进行处理, 主要对 ARP / FDB / Route 进行配置.

源码位置: `pkg/backend/vxlan/vxlan_network.go`

### 创建 vxlan network 实例

flannel 中每个 backend 对象都需要实现 `RegisterNetwork` 方法. 这里举例 `vxlan` 类型的 backend.

`RegisterNetwork` 其内部流程如下.

1. 创建 vxlan 设备, 其内部为幂等的, 其内部过程会解决冲突问题. 
2. 调用 subnet manager 的 AcquireLease 接口完成 lease 地址配置租约的申请.
3. 为 flannel device 设备添加 ip 地址, 子网为 32. 通常地址该 cidr 的网络地址.
4. 创建 network, 内部会监听集群变化, 按照变化事件做出增删改 vxlan 操作.

```go
func (be *VXLANBackend) RegisterNetwork(ctx context.Context, wg *sync.WaitGroup, config *subnet.Config) (backend.Network, error) {
	cfg := struct {
		VNI           int
		Port          int
		GBP           bool
		Learning      bool
		DirectRouting bool
	}{
		VNI: defaultVNI,
	}

	// backend 为空必然是异常的.
	if len(config.Backend) > 0 {
		if err := json.Unmarshal(config.Backend, &cfg); err != nil {
			return nil, fmt.Errorf("error decoding VXLAN backend config: %v", err)
		}
	}

	var dev, v6Dev *vxlanDevice
	var err error
	if config.EnableIPv4 {
		devAttrs := vxlanDeviceAttrs{
			// ...
		}

		// 创建 vxlan 设备, 其内部为幂等的, 其内部过程会解决冲突问题.
		dev, err = newVXLANDevice(&devAttrs)
		if err != nil {
			return nil, err
		}
		dev.directRouting = cfg.DirectRouting
	}
	if config.EnableIPv6 {
		// ...
	}

	// 构建 subnetAttrs 对象, 后用来申请租约.
	subnetAttrs, err := newSubnetAttrs(be.extIface.ExtAddr, be.extIface.ExtV6Addr, uint16(cfg.VNI), dev, v6Dev)
	if err != nil {
		return nil, err
	}

	// 调用 subnet manager 的 AcquireLease 接口完成 lease 地址配置租约的申请.
	lease, err := be.subnetMgr.AcquireLease(ctx, subnetAttrs)
	switch err {
	case nil:
	case context.Canceled, context.DeadlineExceeded:
		// 超时或关闭
		return nil, err
	default:
		return nil, fmt.Errorf("failed to acquire lease: %v", err)
	}

	if config.EnableIPv4 {
		// 获取 flannel 地址信息.
		net, err := config.GetFlannelNetwork(&lease.Subnet)
		if err != nil {
			return nil, err
		}
		// 为 flannel device 设备添加 ip 地址, 子网为 32. 通常地址该 cidr 的网络地址.
		if err := dev.Configure(ip.IP4Net{IP: lease.Subnet.IP, PrefixLen: 32}, net); err != nil {
			return nil, fmt.Errorf("failed to configure interface %s: %w", dev.link.Attrs().Name, err)
		}
	}
	if config.EnableIPv6 {
		// ...
	}

	// 这个重要. 创建 vxlan network, 内部会监听集群变化, 按照变化事件做出增删改 vxlan 操作.
	return newNetwork(be.subnetMgr, be.extIface, dev, v6Dev, ip.IP4Net{}, lease)
}
```


### vxlan network 启动入口

```go
func (nw *network) Run(ctx context.Context) {
	wg := sync.WaitGroup{}

	events := make(chan []subnet.Event)
	wg.Add(1)
	go func() {
		// 从 kube subnet 组件里监听集群内所有 node 的网络变化.
		// 收到变更事件后, 传入 evnets 管道中.
		subnet.WatchLeases(ctx, nw.subnetMgr, nw.SubnetLease, events)
		wg.Done()
	}()

	defer wg.Wait()

	for {
		// 获取来自 informer node 资源变化.
		evtBatch, ok := <-events
		if !ok {
			return
		}

		// 进行 vxlan 网络配置. 主要为添加 arp, fdb, route 等过程.
		nw.handleSubnetEvents(evtBatch)
	}
}
```

### vxlan 的配置实现原理

`handleSubnetEvents` 为配置 vlan 虚拟网络的核心方法, 实现原理是根据传入的事件和配置来添加或删除 vxlan 的 arp, fdb, route 配置.

其详细实现过程看下面代码中注释. 

```go
func (nw *network) handleSubnetEvents(batch []subnet.Event) {
	for _, event := range batch {
		sn := event.Lease.Subnet
		v6Sn := event.Lease.IPv6Subnet
		attrs := event.Lease.Attrs

		// 只处理 vxlan 类型, 其他类型直接跳过.
		if attrs.BackendType != "vxlan" {
			continue
		}

		var (
			vxlanAttrs, v6VxlanAttrs           vxlanLeaseAttrs
			directRoutingOK, v6DirectRoutingOK bool
			directRoute, v6DirectRoute         netlink.Route
			vxlanRoute, v6VxlanRoute           netlink.Route
		)

		if event.Lease.EnableIPv4 && nw.dev != nil {
			// 获取属性信息
			if err := json.Unmarshal(attrs.BackendData, &vxlanAttrs); err != nil {
				continue
			}

			// 构建 vxlan 路由
			vxlanRoute = netlink.Route{
				LinkIndex: nw.dev.link.Attrs().Index,
				Scope:     netlink.SCOPE_UNIVERSE,
				Dst:       sn.ToIPNet(),
				Gw:        sn.IP.ToIP(),
			}
			vxlanRoute.SetFlag(syscall.RTNH_F_ONLINK)

			// 创建直接路由
			directRoute = netlink.Route{
				Dst: sn.ToIPNet(),
				Gw:  attrs.PublicIP.ToIP(),
			}
			if nw.dev.directRouting {
				if dr, err := ip.DirectRouting(attrs.PublicIP.ToIP()); err != nil {
					log.Error(err)
				} else {
					directRoutingOK = dr
				}
			}
		}

		// ...

		switch event.Type {

		// 当事件类型为 add 时, 进行配置该子网路由.
		case subnet.EventAdded: 
			if event.Lease.EnableIPv4 {
				if directRoutingOK {
					if err := netlink.RouteReplace(&directRoute); err != nil {
						continue
					}
				} else {
					// 添加 arp 配置
					if err := nw.dev.AddARP(neighbor{IP: sn.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						continue
					}

					// 添加 fdb 配置
					if err := nw.dev.AddFDB(neighbor{IP: attrs.PublicIP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						// 如果发生异常则回滚删掉 arp.
						if err := nw.dev.DelARP(neighbor{IP: event.Lease.Subnet.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						}

						continue
					}

					// 添加和更新 vxlan route 路由配置.
					if err := netlink.RouteReplace(&vxlanRoute); err != nil {
						// 如果发生失败, 则尝试回收 arp 配置.
						if err := nw.dev.DelARP(neighbor{IP: event.Lease.Subnet.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						}

						// 如果发生失败, 则尝试回收 fdb 配置.
						if err := nw.dev.DelFDB(neighbor{IP: event.Lease.Attrs.PublicIP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						}

						continue
					}
				}
			}
			if event.Lease.EnableIPv6 {
				// ...
			}
		case subnet.EventRemoved:
			if event.Lease.EnableIPv4 {
				if directRoutingOK {
					// 直接路由模式, 只需要删除路由，无需删除 arp 和 fdb.
					if err := netlink.RouteDel(&directRoute); err != nil {
						// ...
					}
				} else {
					// 先删除 arp 配置
					if err := nw.dev.DelARP(neighbor{IP: sn.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						log.Error("DelARP failed: ", err)
					}

					// 再删除 fdb 配置
					if err := nw.dev.DelFDB(neighbor{IP: attrs.PublicIP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						log.Error("DelFDB failed: ", err)
					}

					// 删除 vxlan 路由信息.
					if err := netlink.RouteDel(&vxlanRoute); err != nil {
						log.Errorf("failed to delete vxlanRoute (%s -> %s): %v", vxlanRoute.Dst, vxlanRoute.Gw, err)
					}
				}
			}
			if event.Lease.EnableIPv6 {
				// ...
			}
		default:
			// 非法事件, 当前的代码不会跳到这里.
			log.Error("internal error: unknown event type: ", int(event.Type))
		}
	}
}
```

## host-gw network 的实现原理

下图为 host-gw 跨主机的网络通信原理.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302050019035.png)

在 flannel host-gw 网络模式下, 不涉及 VXLAN 的封包解包, 不需要经过 flannel.1 虚机网卡. flanneld 负责为各节点设置路由, 将对应节点 Pod 子网的下一跳地址指向对应的节点的IP.

源码位置: `pkg/backend/route_network.go`

### host-gw network 启动入口

`Run()` 方法会获取和监听 leases 对象, 并调用 `handleSubnetEvents` 来处理 host-gw 路由配置.

```go
func (n *RouteNetwork) Run(ctx context.Context) {
	wg := sync.WaitGroup{}

	evts := make(chan []subnet.Event)
	wg.Add(1)
	go func() {
		// 从 etcd 或者 k8s apiserver 监听 node lease.
		subnet.WatchLeases(ctx, n.SM, n.SubnetLease, evts)
		wg.Done()
	}()

	n.routes = make([]netlink.Route, 0, 10)
	wg.Add(1)
	go func() {
		// 定时检查注册的路由跟实际 node 路由配置是否有缺失, 不存在时需要添加.
		n.routeCheck(ctx)
		wg.Done()
	}()

	defer wg.Wait()

	for {
		evtBatch, ok := <-evts
		if !ok {
			return
		}
		// 根据 event 处理路由表
		n.handleSubnetEvents(evtBatch)
	}
}
```

### host-gw 配置路由表的原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302051013266.png)

`handleSubnetEvents` 用来根据 event type 来增减路由表, 其内部使用 `netlink` 的 `RouteAdd` 来添加路由条目, 调动 `RouteDel` 来清理路由.

```go
func (n *RouteNetwork) handleSubnetEvents(batch []subnet.Event) {
	for _, evt := range batch {
		switch evt.Type {
		case subnet.EventAdded:
			// 当收到添加事件时, 在路由表里增加相关路由条目.
			if evt.Lease.EnableIPv4 {
				log.Infof("Subnet added: %v via %v", evt.Lease.Subnet, evt.Lease.Attrs.PublicIP)

				// 根据 lease 来构建 route 对象
				route := n.GetRoute(&evt.Lease)

				// 添加路由表
				routeAdd(route, netlink.FAMILY_V4, n.addToRouteList, n.removeFromV4RouteList)
			}

			if evt.Lease.EnableIPv6 {
				// ipv6 暂时忽略
			}

		case subnet.EventRemoved:
			// 当收到删除事件时, 进行相关路由条目删除.
			// ...

			if evt.Lease.EnableIPv4 {
				log.Info("Subnet removed: ", evt.Lease.Subnet)

				// 根据 lease 构建 route 对象.
				route := n.GetRoute(&evt.Lease)
				// 在 route list 里剔除》
				n.removeFromV4RouteList(*route)

				// 删除该路由.
				if err := netlink.RouteDel(route); err != nil {
					log.Errorf("Error deleting route to %v: %v", evt.Lease.Subnet, err)
				}
			}

			if evt.Lease.EnableIPv6 {
				// ipv6 暂时忽略
			}

		default:
			log.Error("Internal error: unknown event type: ", int(evt.Type))
		}
	}
}
```

`GetRoute` 方法是在 `RegisterNetwork` 时赋值的匿名方法, 其内部会把传入的 lease 结构转换为 netlink.Route 路由结构. route 的 dst 字段为 subnet 地址段, gw 是路由的下一条地址, 这里其实就是该 subset 对应的 node 的地址.

```go
n.GetRoute = func(lease *subnet.Lease) *netlink.Route {
	return &netlink.Route{
		Dst:       lease.Subnet.ToIPNet(),       // 目标地址
		Gw:        lease.Attrs.PublicIP.ToIP(),  // 路由下一条地址
		LinkIndex: n.LinkIndex,
	}
}
```

`routeAdd` 用来添加路由条目, 如果该路由条目存在时, 需要先剔除再添加.

```go
func routeAdd(route *netlink.Route, ipFamily int, addToRouteList, removeFromRouteList func(netlink.Route)) {
	addToRouteList(*route)
	// 先检查是否存在.
	routeList, err := netlink.RouteListFiltered(ipFamily, &netlink.Route{Dst: route.Dst}, netlink.RT_FILTER_DST)
	if err != nil {
		log.Warningf("Unable to list routes: %v", err)
	}

	// 如果存在且配置不一致时, 需要先删除路由条目.
	if len(routeList) > 0 && !routeEqual(routeList[0], *route) {
		if err := netlink.RouteDel(&routeList[0]); err != nil {
			return
		}
		removeFromRouteList(routeList[0])
	}

	// 再次获取跟传入的目标 dst 一致的路由列表.
	routeList, err = netlink.RouteListFiltered(ipFamily, &netlink.Route{Dst: route.Dst}, netlink.RT_FILTER_DST)
	if err != nil {
		log.Warningf("Unable to list routes: %v", err)
	}

	// 有路由表, 而且配置一致, 则忽略, 否则进行添加路由表.
	if len(routeList) > 0 && routeEqual(routeList[0], *route) {
		log.Infof("Route to %v already exists, skipping.", route)

	} else if err := netlink.RouteAdd(route); err != nil {
		log.Errorf("Error adding route to %v: %s", route, err)
		return
	}

	// 是否可以正常获取对应条件的路由表, 不理解为什么又要判断. 😅
	_, err = netlink.RouteListFiltered(ipFamily, &netlink.Route{Dst: route.Dst}, netlink.RT_FILTER_DST)
	if err != nil {
		log.Warningf("Unable to list routes: %v", err)
	}
}
```

`host-gw` 的路由配置原理还是比较简单的. 首先读取整个集群里所有的 node 的 ip 和 cidr 子网等信息, 然后设置对应的路由策略. 在一个 node 节点上含有整个集群的路由信息.

```
10.230.10.0/24 dev cni0 proto kernel scope link src 10.230.10.1

10.244.20.0/24 via 172.16.0.102 dev eth0
10.244.21.0/24 via 172.16.0.121 dev eth0
10.244.22.0/24 via 172.16.0.122 dev eth0
10.244.23.0/24 via 172.16.0.123 dev eth0
10.244.24.0/24 via 172.16.0.124 dev eth0
10.244.25.0/24 via 172.16.0.125 dev eth0
...
```

### 定时修复路由表

`routeCheck` 会周期性的修复本地的路由表, 检测对比当前主机跟内存路由表, 如有缺失则添加路由规则. 

**为什么需要周期性的修复路由表 ?**

1. 某个路由表的规则被手动删除 ? 正常没这个可能.
2. `handleSubnetEvents` 在添加路由表时, 对失败的 `netlink.RouteAdd` 没有重试, 所以在失败后进行周期性的重试.

```go
const (
	routeCheckRetries = 10
)

// 周期检查路由表是否有缺失, 有缺失下进行添加路由表.
func (n *RouteNetwork) routeCheck(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case <-time.After(routeCheckRetries * time.Second):
			// 每次间隔 10s 秒.
			n.checkSubnetExistInV4Routes()
			n.checkSubnetExistInV6Routes()
		}
	}
}

func (n *RouteNetwork) checkSubnetExistInV4Routes() {
	n.checkSubnetExistInRoutes(n.routes, netlink.FAMILY_V4)
}

func (n *RouteNetwork) checkSubnetExistInV6Routes() {
	n.checkSubnetExistInRoutes(n.v6Routes, netlink.FAMILY_V6)
}

func (n *RouteNetwork) checkSubnetExistInRoutes(routes []netlink.Route, ipFamily int) {
	// 首先通过 netlink 获取当前节点上的所有路由表
	routeList, err := netlink.RouteList(nil, ipFamily)
	if err == nil {
		// 内存里的配置跟当前的节点配置做遍历对比.
		for _, route := range routes {
			exist := false
			for _, r := range routeList {
				if r.Dst == nil {
					// 跳过
					continue
				}
				if routeEqual(r, route) {
					// 一致则标记已存在
					exist = true
					break
				}
			}

			// 如不存在, 则进行路由表添加, 其实是个修复的过程.
			if !exist {
				if err := netlink.RouteAdd(&route); err != nil {
					continue
				} else {
					log.Infof("Route recovered %v : %v", route.Dst, route.Gw)
				}
			}
		}
	} else {
		log.Errorf("Error fetching route list. Will automatically retry: %v", err)
	}
}
```

### host-gw 适用于二层网络

flannal host-gw 方案当前只适用于同一个二层网络下, node 之间需要在一个 vlan 里, 因为 host-gw 在数据链路层会把目标的 MAC 地址换成目标 node 上的 MAC. 但如果两个 node 在不同的 vlan 虚拟局域网里, 那么由于 vlan 会隔离抑制广播, 通过 arp 广播自然无法拿到目标的 mac 网卡地址, 自然就无法把数据报文发出去.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302052018408.png)

## 总结

下图 subnet manager 使用 k8s kube-apiserver, backend 选用 vxlan 网络.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302041931615.png)

下图 subnet manager 使用 k8s kube-apiserver, backend 选用 host-gw 网络.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202302/202302051020026.png)

flannel 的实现原理大体分析完了, 其原理就是监听 etcd 或者 k8s apiserver 的 node 对象, 从 node 资源对象中解析到 spec.PodCIDR 等字段来构建 lease 对象. 根据 backend 的类型构建不同的 network 对象. 

network 控制器从 subnet manager 监听获取 lease 对象, 然后配置网络. vxlan network 则需要进行 ARP, FDB, Route 配置流程, 而 host-gw 只需配置路由 route 规则即可.

本文主要分析的 flannel 的代码实现过程, 后面会跟进继续分析 calico 和 cilium 的实现原理, 请关注订阅 [http://github.com/rfyiamcool/notes](http://github.com/rfyiamcool/notes)