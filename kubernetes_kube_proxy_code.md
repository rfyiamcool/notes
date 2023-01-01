## 源码分析 kubernetes kube-proxy 的实现原理

`kubernetes kube-proxy 的版本是 v1.27.0`

### cmd 入口

`kube-proxy` 启动的过程就不细说了，简单说就是解析传递配置，构建 `ProxyServer` 服务，最后启动 `ProxyServer`.

```js
cmd/kube-proxy/proxy.go: main
    cmd/kube-proxy/app/server.go: NewProxyCommand
        cmd/kube-proxy/app/server.go: ProxyServer.Run()
```

### ProxyServer

在创建 `NewProxyServer()` 时，根据 `proxyMode` 实例化不同的 proxier 对象. 在 kube-proxy 里 proxier 有iptables 和 ipvs 两种实现, 以前在 userspace 实现的代理已经在老版本中剔除了.

代码位置: `cmd/kube-proxy/app/server.go`

```go
// NewProxyServer returns a new ProxyServer.
func NewProxyServer(o *Options) (*ProxyServer, error) {
	return newProxyServer(o.config, o.master)
}

func newProxyServer(
	config *proxyconfigapi.KubeProxyConfiguration,
	master string) (*ProxyServer, error) {
    
	// 实例化 iptables 的 proxy
	if proxyMode == proxyconfigapi.ProxyModeIPTables {
		if dualStack {
		} else {
			proxier, err = iptables.NewProxier(
				iptInterface,
				utilsysctl.New(),
				execer,
				config.IPTables.SyncPeriod.Duration,
				config.IPTables.MinSyncPeriod.Duration,
				...
			)
		}

	// 实例化 ipvs 的 proxy
	} else if proxyMode == proxyconfigapi.ProxyModeIPVS {
		// 实例化 kernel, ipset，ipvs 管理工具
		kernelHandler := ipvs.NewLinuxKernelHandler()
		ipsetInterface = utilipset.New(execer)
		ipvsInterface = utilipvs.New()

		if dualStack {
		} else {
			if err != nil {
				return nil, fmt.Errorf("unable to create proxier: %v", err)
			}

			proxier, err = ipvs.NewProxier(
				iptInterface,
				ipvsInterface,
				ipsetInterface,
				utilsysctl.New(),
				...
			)
		}
		...
	}

	return &ProxyServer{
		Client:                 client,
		EventClient:            eventClient,
		IptInterface:           iptInterface,
		IpvsInterface:          ipvsInterface,
		IpsetInterface:         ipsetInterface,
		...
	}, nil
}
```

`ProxyServer` 在调用 Run() 启动时会实例化 informer 监听器，然后一直监听 apiserver 反馈的 service, endpoints, node 变动事件.

```go
func (s *ProxyServer) Run() error {
	informerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.LabelSelector = labelSelector.String()
		}))

	// 监听 service resource 资源
	serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.Proxier)
	go serviceConfig.Run(wait.NeverStop)

	// 监听 endpoints resource 资源
	endpointSliceConfig := config.NewEndpointSliceConfig(informerFactory.Discovery().V1().EndpointSlices(), s.ConfigSyncPeriod)
	endpointSliceConfig.RegisterEventHandler(s.Proxier)
	go endpointSliceConfig.Run(wait.NeverStop)

	informerFactory.Start(wait.NeverStop)

	// 监听 node 资源更新
	currentNodeInformerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.FieldSelector = fields.OneTermEqualSelector("metadata.name", s.NodeRef.Name).String()
		}))
	nodeConfig := config.NewNodeConfig(currentNodeInformerFactory.Core().V1().Nodes(), s.ConfigSyncPeriod)
	nodeConfig.RegisterEventHandler(s.Proxier)
	go nodeConfig.Run(wait.NeverStop)

	currentNodeInformerFactory.Start(wait.NeverStop)

	s.birthCry()

	go s.Proxier.SyncLoop()

	return <-errCh
}
```

### 如何监听和处理事件 ? 

kube-proxy 开启了对 services 和 endpoints 的事件监听，但篇幅原因这里就只聊 Services 的事件监听.

初始化阶段时会实例化 ServiceConfig 对象， 并向 serviceConfig 注册 proxier 对象. `ServiceConfig` 实现了对 informer 的监听，并向 informer 注册封装的回调接口. 当有 service 的增删改事件时, 调用 proxier 的 `OnServiceAdd, OnServiceUpdate, OnServiceDelete` 方法.

```go
type ServiceConfig struct {
	listerSynced  cache.InformerSynced
	eventHandlers []ServiceHandler
}

// NewServiceConfig creates a new ServiceConfig.
func NewServiceConfig(serviceInformer coreinformers.ServiceInformer, resyncPeriod time.Duration) *ServiceConfig {
	result := &ServiceConfig{
		listerSynced: serviceInformer.Informer().HasSynced,
	}

	serviceInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    result.handleAddService,
			UpdateFunc: result.handleUpdateService,
			DeleteFunc: result.handleDeleteService,
		},
		resyncPeriod,
	)

	return result
}

// 注册事件处理，其实就是 proxier, 它实现了 ServiceHandler 接口.
func (c *ServiceConfig) RegisterEventHandler(handler ServiceHandler) {
	c.eventHandlers = append(c.eventHandlers, handler)
}

func (c *ServiceConfig) Run(stopCh <-chan struct{}) {
	// 第一次启动时, 同步下数据到缓存
	if !cache.WaitForNamedCacheSync("service config", stopCh, c.listerSynced) {
		return
	}

	// proxier 的 OnServiceSynced 实现是调用 `proxier.syncProxyRules()`
	for i := range c.eventHandlers {
		c.eventHandlers[i].OnServiceSynced()
	}
}

// 处理 service 新增事件
func (c *ServiceConfig) handleAddService(obj interface{}) {
	service, ok := obj.(*v1.Service)
	for i := range c.eventHandlers {
		c.eventHandlers[i].OnServiceAdd(service)
	}
}

// 处理 service 更新事件
func (c *ServiceConfig) handleUpdateService(oldObj, newObj interface{}) {
	oldService, ok := oldObj.(*v1.Service)
	for i := range c.eventHandlers {
		c.eventHandlers[i].OnServiceUpdate(oldService, service)
	}
}

// 处理 service 删除事件
func (c *ServiceConfig) handleDeleteService(obj interface{}) {
	service, ok := obj.(*v1.Service)
	for i := range c.eventHandlers {
		c.eventHandlers[i].OnServiceDelete(service)
	}
}
```

从 ipvs 的 proxier 里, 分析下 `OnServiceAdd`, `OnServiceUpdate`, `OnServiceDelete` 这几个方法的实现. 其实就是对 proxier 的 `serviceChanges` 做存储变动对象操作. 需要注意的是，真正处理变动的在 `proxier.syncProxyRules()` 方法里.

```go
// OnServiceAdd is called whenever creation of new service object is observed.
func (proxier *Proxier) OnServiceAdd(service *v1.Service) {
	proxier.OnServiceUpdate(nil, service)
}

// OnServiceUpdate is called whenever modification of an existing service object is observed.
func (proxier *Proxier) OnServiceUpdate(oldService, service *v1.Service) {
	if proxier.serviceChanges.Update(oldService, service) && proxier.isInitialized() {
		proxier.Sync()
	}
}

// OnServiceDelete is called whenever deletion of an existing service object is observed.
func (proxier *Proxier) OnServiceDelete(service *v1.Service) {
	proxier.OnServiceUpdate(service, nil)
}
```

### 真正干活 Proxier 的实现

在阅读了 kube-proxy 源码后可以发现，Proxier 才是真正干活的类，kube-proxy 里对于 ipvs，iptables, ipset 等工具的操作都在 proxier 里.

在 kube-proxy 中 proxier 有 iptables 和 ipvs 两种实现, 现在已经少有用 iptables 的了，所以就只分析 ipvs proxier 实现.

在实例化 `Proxier` 对象时会注册一个定时任务，它会周期性调用 `syncProxyRules` 方法. 

```go
func NewProxier(
	syncPeriod time.Duration,
	minSyncPeriod time.Duration,
	masqueradeAll bool,
	...
) (*Proxier, error) {
	...
	proxier := &Proxier{
		svcPortMap:            make(proxy.ServicePortMap),
		endpointsMap:          make(proxy.EndpointsMap),
		...
	}

	...
	proxier.syncRunner = async.NewBoundedFrequencyRunner("sync-runner", proxier.syncProxyRules, minSyncPeriod, syncPeriod, burstSyncs)
	return proxier, nil
}
```

`syncProxyRules` 函数是 `kube-proxy` 实现的核心. 该主要逻辑每次都从头开始遍历 servcie 和 endpoints 所有对象，对比当前配置来进行增删改 ipvs virtualserver 和 realserver, 及绑定解绑 dummy ip地址，还有配置 ipset 和 iptables 的相关策略.

代码位置: [https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/proxier.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/proxier.go)

```go
func (proxier *Proxier) syncProxyRules() {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()
    
	// 把从 informer 拿到变更的 servcie 结构，更新到 svcPortMap 里, 每次 Update 还会把 changes 清空.
	serviceUpdateResult := proxier.svcPortMap.Update(proxier.serviceChanges)
    
	// 把从 informer 拿到变更的 endpoint 结构，更新到 endpointsMap 里, 每次 Update 还会把 changes 清空.
	endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)

	// 创建名为 `kube-ipvs0` 的 dummy 类型网络设备
	_, err := proxier.netlinkHandle.EnsureDummyDevice(defaultDummyDevice)
	if err != nil {
		return
	}

	// 创建 ipset 规则，调用 ipset create 创建 ipset 时，还需指定 size 和 entry 的格式.
	for _, set := range proxier.ipsetList {
		if err := ensureIPSet(set); err != nil {
			return
		}
		set.resetEntries()
	}

	// 遍历当前的 services 对象集合
	for svcPortName, svcPort := range proxier.svcPortMap {
		// 遍历 endpoints 对象
		for _, e := range proxier.endpointsMap[svcPortName] {

			// 拿到 endpoints 对象
			ep, ok := e.(*proxy.BaseEndpointInfo)
			epIP := ep.IP()
			epPort, err := ep.Port()
            
			// 定义 endpoint 的 ipset entry 结构
			entry := &utilipset.Entry{
				IP:       epIP,
				Port:     epPort,
				Protocol: protocol,
				IP2:      epIP,
				SetType:  utilipset.HashIPPortIP,
			}
            
        		// 在 kubeLoopBackIPSet 配置集合中加入 entry 配置.
			proxier.ipsetList[kubeLoopBackIPSet].activeEntries.Insert(entry.String())
		}

		// 定义 service 的 ipset entry 结构
		entry := &utilipset.Entry{
			IP:       svcInfo.ClusterIP().String(),
			Port:     svcInfo.Port(),
		}

		// 把 service ipset entry 加入到 kubeClusterIPSet 配置集合中
		proxier.ipsetList[kubeClusterIPSet].activeEntries.Insert(entry.String())
		serv := &utilipvs.VirtualServer{
			Address:   svcInfo.ClusterIP(),
			Port:      uint16(svcInfo.Port()),
		}

		// 创建或更新 lvs virtualServer 配置
		if err := proxier.syncService(svcPortNameString, serv, true, bindedAddresses); err == nil {
			// 创建或更新 lvs realserver 配置
			if err := proxier.syncEndpoint(svcPortName, internalNodeLocal, serv); err != nil {}
		}
	}
    
	// 篇幅不提, 解决 external ip.
	for _, externalIP := range svcInfo.ExternalIPStrings() {
		...
	}
	// 篇幅不提，解决 load balancer
	for _, ingress := range svcInfo.LoadBalancerIPStrings() {
		...
	}

	// 同步 ipset 配置.
	for _, set := range proxier.ipsetList {
		set.syncIPSetEntries()
	}

	// 同步 iptables 的配置.
	proxier.writeIptablesRules()
	proxier.iptablesData.Reset()
	proxier.iptablesData.Write(proxier.natChains.Bytes())
	proxier.iptablesData.Write(proxier.natRules.Bytes())
    err = proxier.iptables.RestoreAll(proxier.iptablesData.Bytes(), utiliptables.NoFlushTables, utiliptables.RestoreCounters)
	...

	// 清理绑定的ip地址
	legacyBindAddrs := proxier.getLegacyBindAddr(activeBindAddrs, currentBindAddrs)

	// 清理需要删除 service, 逻辑里含有 ipvs vs 和 rs 的清理.
	proxier.cleanLegacyService(activeIPVSServices, currentIPVSServices, legacyBindAddrs)

	// 遍历不新鲜的 servcies 集合，通过 contrack tool 工具剔除在 contrack 里协议为 UDP 的旧连接.
	for _, svcIP := range staleServices.UnsortedList() {
		if err := conntrack.ClearEntriesForIP(proxier.exec, svcIP, v1.ProtocolUDP); err != nil {
			klog.ErrorS(err, "Failed to delete stale service IP connections", "IP", svcIP)
		}
	}
}
```

#### 同步 ipset 配置 ? (syncIPSetEntries)

`syncIPSetEntries()` 用来同步刷新 ipset 配置，就是把 ip 地址通过 ipset 命令更新到对应的 ipset 集合里.

代码位置: `pkg/proxy/ipvs/ipset.go`

```go
func (set *IPSet) syncIPSetEntries() {
	// 从本机获取 name 相关所有的 ipset entry
	appliedEntries, err := set.handle.ListEntries(set.Name)
	if err != nil {
		return
	}

	// 放到一个 string set 集合里，为后面做差集做准备
	currentIPSetEntries := sets.NewString()
	for _, appliedEntry := range appliedEntries {
		currentIPSetEntries.Insert(appliedEntry)
	}

	// 如果不相等的话, 进行同步操作.
	if !set.activeEntries.Equal(currentIPSetEntries) {
		// 删除遗留的 ipset 元素. 也就是删除本机有，但在激活集合中不存在的 ipset entry.
		for _, entry := range currentIPSetEntries.Difference(set.activeEntries).List() {
			if err := set.handle.DelEntry(entry, set.Name); err != nil {
				...
			}
		}
		// 添加 ipset 元素. 添加本机没有但在激活集合中存在的 ipset entry.
		for _, entry := range set.activeEntries.Difference(currentIPSetEntries).List() {
			if err := set.handle.AddEntry(entry, &set.IPSet, true); err != nil {
			}
		}
	}
}
```

ipset 的最终是依赖 exec 去调用 ipset 二进制执行文件实现的. 增删改 ipset 集合时先是拼凑 ipset 命令的参数，然后在传递给 ipset 命令去执行.

代码位置: `pkg/util/ipset/ipset.go`

```go
const IPSetCmd = "ipset"

type IPSet struct {
	Name string
	SetType Type
	HashSize int
	MaxElem int
	...
}

func (runner *runner) AddEntry(entry string, set *IPSet, ignoreExistErr bool) error {
	args := []string{"add", set.Name, entry}
	if ignoreExistErr {
		args = append(args, "-exist")
	}
	if out, err := runner.exec.Command(IPSetCmd, args...).CombinedOutput(); err != nil {
		return fmt.Errorf("error adding entry %s, error: %v (%s)", entry, err, out)
	}
	return nil
}

func (runner *runner) DelEntry(entry string, set string) error {
	if out, err := runner.exec.Command(IPSetCmd, "del", set, entry).CombinedOutput(); err != nil {
		return fmt.Errorf("error deleting entry %s: from set: %s, error: %v (%s)", entry, set, err, out)
	}
	return nil
}
```

#### 同步 ipvs service 配置 ? (syncService)

如果 ipvs vs 在当前节点配置过，且配置没有变动，另外 service ip 已绑定到 dummy 设备上，就一路绿灯无需真实去操作.

如果当前节点不存在该 `ipvs VirtualServer`, 则需要进行创建，如果已经存在，但配置不同，则进行更新 VirtualServer，另外还需要在本地 `kube-ipvs0` 设备上绑定 service clusterIP 地址.

k8s 的 ipvs 是通过 `moby/ipvs` 库实现增删改查操作, moby/ipvs 里使用 `netlink` 实现用户态进程跟内核态进行通信.

```go
func (proxier *Proxier) syncService(svcName string, vs *utilipvs.VirtualServer, bindAddr bool, bindedAddresses sets.String) error {
	// 从当前节点里获取该 ipvs vs 配置
	appliedVirtualServer, _ := proxier.ipvs.GetVirtualServer(vs)
	// 如果当前节点不存在该 ipvs vs 配置，或者 有配置但配置发生了变更.
	if appliedVirtualServer == nil || !appliedVirtualServer.Equal(vs) {
		// 当前节点不存在该 ipvs vs, 则进行创建
		if appliedVirtualServer == nil {
			klog.V(3).InfoS("Adding new service", "serviceName", svcName, "virtualServer", vs)
			if err := proxier.ipvs.AddVirtualServer(vs); err != nil {
				return err
			}
		} else {
 			// 如果配置发生变更，则进行 update 更新.
			klog.V(3).InfoS("IPVS service was changed", "serviceName", svcName)
			if err := proxier.ipvs.UpdateVirtualServer(vs); err != nil {
				return err
			}
		}
	}

	// 把 service ip 绑定到 dummy 设备上，也就是 kube-ipvs0 设备.
	if bindAddr {
		// 如果已经绑定了，那就无需再绑定.
		if bindedAddresses != nil && bindedAddresses.Has(vs.Address.String()) {
			return nil
		}
        
		// 绑定 ip 地址, 重复执行也不会返回 error，可理解为幂等操作.
		_, err := proxier.netlinkHandle.EnsureAddressBind(vs.Address.String(), defaultDummyDevice)
		if err != nil {
			return err
		}
	}

	return nil
}
```

#### 如何删除 ipvs service ? (cleanLegacyService)

对比 active 和主机的 ipvs vs 的配置，如果主机存在某 service 配置，但 active 里没有，则删除并进行解绑 ip.

```go
func (proxier *Proxier) cleanLegacyService(activeServices map[string]bool, currentServices map[string]*utilipvs.VirtualServer, legacyBindAddrs map[string]bool) {
	isIPv6 := netutils.IsIPv6(proxier.nodeIP)
	// 使用当前的 services 进行遍历
	for cs := range currentServices {
		svc := currentServices[cs]
		...
		// 如果激活集合里存在，但当前集合不存在，说明需要删除该 service.
		if _, ok := activeServices[cs]; !ok {
			klog.V(4).InfoS("Delete service", "virtualServer", svc)
			// 调用 ipvs 删除 svc 相关的负载策略
			if err := proxier.ipvs.DeleteVirtualServer(svc); err != nil {
				klog.ErrorS(err, "Failed to delete service", "virtualServer", svc)
			}
			addr := svc.Address.String()
			if _, ok := legacyBindAddrs[addr]; ok {
				klog.V(4).InfoS("Unbinding address", "address", addr)
				// 解绑 cluster ip 地址
				proxier.netlinkHandle.UnbindAddress(addr, defaultDummyDevice)
				delete(legacyBindAddrs, addr)
			}
		}
	}
}
```

#### 同步 ipvs endpoints 配置 (syncEndpoint)?

对比当前 virtualServer 的 realserver 集合，如果有新增变动，则调用 ipvs.AddRealServer, 有变动则使用 UpdateRealServer 更改.

```go
func (proxier *Proxier) syncEndpoint(svcPortName proxy.ServicePortName, onlyNodeLocalEndpoints bool, vs *utilipvs.VirtualServer) error {
	// 获取当前主机的 ipvs vs 配置
	appliedVirtualServer, err := proxier.ipvs.GetVirtualServer(vs)
	if err != nil {
		klog.ErrorS(err, "Failed to get IPVS service")
		return err
	}

	curEndpoints := sets.NewString()
	curDests, err := proxier.ipvs.GetRealServers(appliedVirtualServer)
	if err != nil {
		return err
	}

	// 把 realserver 放到一个 set string 集合里
	for _, des := range curDests {
		curEndpoints.Insert(des.String())
	}

	endpoints := proxier.endpointsMap[svcPortName]
	newEndpoints := sets.NewString()
	for _, epInfo := range endpoints {
		newEndpoints.Insert(epInfo.String())
	}

	// 添加 realserver
	for _, ep := range newEndpoints.List() {
		ip, port, err := net.SplitHostPort(ep)
		portNum, err := strconv.Atoi(port)
		newDest := &utilipvs.RealServer{
			Address: netutils.ParseIPSloppy(ip),
			Port:    uint16(portNum),
			Weight:  1,
		}

		// 更新 realServer
		if curEndpoints.Has(ep) {
			if proxier.initialSync {
				for _, dest := range curDests {
					// 如果新旧的权重不同, 则调用 ipvs 接口进行变更
					if dest.Weight != newDest.Weight {
						err = proxier.ipvs.UpdateRealServer(appliedVirtualServer, newDest)
						if err != nil {
							continue
						}
					}
				}
			}

			...
		}
		// 为 vs 添加新的 realServer
		err = proxier.ipvs.AddRealServer(appliedVirtualServer, newDest)
		if err != nil {
			klog.ErrorS(err, "Failed to add destination", "newDest", newDest)
			continue
		}
	}
	...
	return nil
}
```

#### 在 conntrack 里剔除 UDP 旧连接 (ClearEntriesForIP) ?

kube-proxy 在删除 UDP 的 service 时候, 也需要清除这些 ip_conntrack 连接，否则后面的流量还是会被导入的废弃的 pod 上面. 为什么 TCP 不需要在 conntrack 里剔除，而 UDP 需要 ? 这是因为 TCP 本是有状态的连接，当把流量切到其他 pod 时，旧连接会因为对端在关闭时进行挥手 fin，或者不可达时，自然就会开辟新连接，新连接自然使用新的策略. UDP 由于是无状态的连接，所以不受影响, 虽然数据不可达，但照旧能用，所以需要剔除旧的 UDP conntrack 状态.

剔除连接的原理很简单，就是调用 conntrack-tools 来剔除连接，我们也可以使用 conntrack 工具来手动剔除.

比如 `conntrack -D -s 114.114.114.114 -p UDP`.

```go
func ClearEntriesForIP(execer exec.Interface, ip string, protocol v1.Protocol) error {
	parameters := parametersWithFamily(utilnet.IsIPv6String(ip), "-D", "--orig-dst", ip, "-p", protoStr(protocol))
	err := Exec(execer, parameters...)
	if err != nil && !strings.Contains(err.Error(), NoConnectionToDelete) {
		return fmt.Errorf("error deleting connection tracking state for UDP service IP: %s, error: %v", ip, err)
	}
	return nil
}

func Exec(execer exec.Interface, parameters ...string) error {
	conntrackPath, err := execer.LookPath("conntrack")
	if err != nil {
		return fmt.Errorf("error looking for path of conntrack: %v", err)
	}

	output, err := execer.Command(conntrackPath, parameters...).CombinedOutput()
	return nil
}
```

### ipvs 里 iptables 相关配置

#### 分析问题

ipvs 里主要的 iptables 的实现都在 `	proxier.writeIptablesRules()` 方法里，代码本不复杂，就是一些 iptables 的规则拼凑，所以不从源码来解析，换个角度直接通过生成的 iptabels 和 chains 流程走向来分析.

另外, kubernetes 在 ipvs 的 README.md 里有详细的 ipvs 和 iptables 的配置, 这里只说 `--cluster-cidr=<cidr>`, 暂时不深入 `--masquerade-all=true` 的配置.

代码位置: [https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md)

#### iptables chains 流程走向

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212072130664.png)

#### iptables 配置如下:

```js
# iptables -t nat -nL

Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
KUBE-POSTROUTING  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */

Chain KUBE-MARK-MASQ (3 references)
target     prot opt source               destination
MARK       all  --  0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-POSTROUTING (1 references)
target     prot opt source               destination
MASQUERADE  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ mark match 0x4000/0x4000
MASQUERADE  all  --  0.0.0.0/0            0.0.0.0/0            match-set KUBE-LOOP-BACK dst,dst,src

Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  -- !10.244.16.0/24       0.0.0.0/0            match-set KUBE-CLUSTER-IP dst,dst
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            match-set KUBE-CLUSTER-IP dst,dst
```