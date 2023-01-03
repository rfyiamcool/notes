## 源码分析 kubernetes coredns 插件开发和服务发现的实现原理

Kubernetes Coredns controller 控制器是用来监听 kube-apiserver 获取 service 等资源配置, 资源发生变更时修改内存里的缓存. 当 coredns server 收到 dns query 请求时, 根据请求的域名在本地缓存 indexer 中找到对象, 组装记录后并返回, 完成域名解析.

k8s coredns controller 是已 coredns plugin 插件形式存在的, 插件的设计也是 coredns 一个优势所在.

**coredns 项目地址:**

从 kubernetes v1.11 起，coreDNS 已经取代 kube-dns 成为默认的 DNS 方案.

[https://github.com/coredns](https://github.com/coredns)

**coredns controller 调用关系**

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301022021998.png)

### coredns plugin 的实现原理

#### 加载插件

这里用 log 插件举例说明, 在 `init()` 内注册 log 的 setup 装载方法, 然后使用 AddPlugin 注册一个 plugin.Plugin 方法. 其目的就是要实现中间件那种调用链.

代码位置: `plugin/log/setup.go`

```go
// 注册 log 插件, 把 log 的 setup 装载方法注册进去.
func init() { plugin.Register("log", setup) }

func setup(c *caddy.Controller) error {
	// Logger 实现了 plugin.Handler 接口了, 注册一个 plugin.Plugin 方法.
	dnsserver.GetConfig(c).AddPlugin(func(next plugin.Handler) plugin.Handler {
		return Logger{Next: next, Rules: rules, repl: replacer.New()}
	})

	return nil
}
```

#### 实例化 coredns server

coredns 在创建 Server 时, 倒序遍历已注册的 plugin 来构建 pluginChain 调用链.

```go
func NewServer(addr string, group []*Config) (*Server, error) {
	s := &Server{
		Addr:         addr,
		zones:        make(map[string][]*Config),
		graceTimeout: 5 * time.Second,
		readTimeout:  3 * time.Second,
		writeTimeout: 5 * time.Second,
		...
	}

	...

	for _, site := range group {
		var stack plugin.Handler
		// 倒序插入
		for i := len(site.Plugin) - 1; i >= 0; i-- {
			// 两个邻居的 plugin 关联起来.
			stack = site.Plugin[i](stack)
			site.registerHandler(stack)

			...
		}

		// 赋值 plugin 调用链
		site.pluginChain = stack
	}

	...

	return s, nil
}
```

启动 udp/tcp 的监听, 读取 dns 请求报文, 处理 dns 请求.

- `ServePacket` 会启动启动 dns server, 并启动 udp server ;
- `ActivateAndServe` 会根据配置来启动 tcp 和 udp 服务 ;
- `serveUDP` 读取 dns 请求报文, 然后开一个协程去调用 `serveUDPPacket` ;
- `serveUDPPacket` 内部会实例化 writer 写对象, 然后调用 `serveDNS` 处理请求.

```go
func (s *Server) ServePacket(p net.PacketConn) error {
	s.m.Lock()
	s.server[udp] = &dns.Server{PacketConn: p, Net: "udp", Handler: dns.HandlerFunc(func(w dns.ResponseWriter, r *dns.Msg) {
		// 调用 ServeDns 方法
		s.ServeDNS(ctx, w, r)
	}), TsigSecret: s.tsigSecret}
	s.m.Unlock()

	// 启动监听
	return s.server[udp].ActivateAndServe()
}

// 根据 srv.net 决定是开启 udp 还是 tcp server.
func (srv *Server) ActivateAndServe() error {
	// 启动 udp 服务, 当 srv.net 为 udp 时启动 udp server.
	if srv.PacketConn != nil {
		return srv.serveUDP(srv.PacketConn)
	}

	// 启动 tcp 服务, srv.net 为 tcp 时才启动 tcp server.
	if srv.Listener != nil {
		return srv.serveTCP(srv.Listener)
	}
	return &Error{err: "bad listeners"}
}

// serveUDP starts a UDP listener for the server.
func (srv *Server) serveUDP(l net.PacketConn) error {
	for srv.isStarted() {
		...

		// 读取 dns 请求报文
		if isUDP {
			m, sUDP, err = reader.ReadUDP(lUDP, rtimeout)
		} else {
			m, sPC, err = readerPC.ReadPacketConn(l, rtimeout)
		}

		// 异常处理 dns 请求, 每个请求都会启动一个协会处理.
		go srv.serveUDPPacket(&wg, m, l, sUDP, sPC)
	}

	return nil
}

// 处理 dns 请求
func (srv *Server) serveUDPPacket(wg *sync.WaitGroup, m []byte, u net.PacketConn, udpSession *SessionUDP, pcSession net.Addr) {
	// 构建 response writer
	w := &response{tsigProvider: srv.tsigProvider(), udp: u, udpSession: udpSession, pcSession: pcSession}
	if srv.DecorateWriter != nil {
		w.writer = srv.DecorateWriter(w)
	} else {
		w.writer = w
	}

	// 调用 ServeDNS 处理请求
	srv.serveDNS(m, w)
}
```

#### coredns serveDNS 主处理逻辑

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301022231339.png)

遍历执行 pluginChain 插件链的所有 Plugin 插件, 依次执行 plugin.ServeDNS 方法.

代码位置: `core/dnsserver/server.go`

```go
func (s *Server) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) {
	...

	// 判断 edns 判断, edns 是个很厉害的功能, 没有 edns 之前, gslb 拿到的只是 dns server 地址, 拿不到用户地址. 而使用 edns 开放协议后, gslb 可以拿到用户的地址. 这样的流量调度才更加准确.
	if m, err := edns.Version(r); err != nil { // Wrong EDNS version, return at once.
		w.WriteMsg(m)
		return
	}

	w = request.NewScrubWriter(r, w)

	for {
		// 遍历每个 zone
		if z, ok := s.zones[q[off:]]; ok {
			for _, h := range z {
				...
				if passAllFilterFuncs(ctx, h.FilterFuncs, &request.Request{Req: r, W: w}) {
					if r.Question[0].Qtype != dns.TypeDS {
						// 执行 plugin 插件调用链, 调用每个插件的 ServeDNS 方法.
						rcode, _ := h.pluginChain.ServeDNS(ctx, w, r)
						if !plugin.ClientWrite(rcode) {
						}
						return
					}
					dshandler = h
				}
			}
		}

		off, end = dns.NextLabel(q, off)
		if end {
			break
		}
		...
	}

	...

	errorAndMetricsFunc(s.Addr, w, r, dns.RcodeRefused)
}
```

### 加载 kubernetes 插件

```go
const pluginName = "kubernetes"

// 把插件注册到 caddy 的 plugins 注册表里, 格式为 map[serverType][plugin.name]Plugin
func init() { plugin.Register(pluginName, setup) }

// 解析配置, 生成 kubeclient, 实例化 newdnsController 控制器等
func setup(c *caddy.Controller) error {
	// 解析 k8s 配置
	k, err := kubernetesParse(c)
	if err != nil {
		return plugin.Error(pluginName, err)
	}

	// 实例化 kubeclient 并 实例化 dns controller 控制器
	onStart, onShut, err := k.InitKubeCache(context.Background())
	if err != nil {
		return plugin.Error(pluginName, err)
	}
	...

	// 把插件注册到 plugin 链表里, 执行效果就是 middleware 中间件模型
	dnsserver.GetConfig(c).AddPlugin(func(next plugin.Handler) plugin.Handler {
		k.Next = next
		return k
	})

	// 获取本地非回环的 ip 地址集合
	c.OnStartup(func() error {
		k.localIPs = boundIPs(c)
		return nil
	})

	return nil
}
```

后面补一篇专门说下 coredns server 的实现原理, 其关键几个插件的实现, 另外如何自定义开发插件.

### dnsController 控制器逻辑

源码位置: `plugin/kubernetes/controller.go`

#### 实例化控制器

实例化 dnsControl 对象, 同时实例化各个资源的 informer 对象, 在实例化 informer 时传入自定义的 eventHandler 回调方法 和 indexer 索引方法.

- 为什么需要监听 service 资源 ? 用户在 pod 内直接使用 service_name 获取到 cluster ip.

- 为什么需要监听 pod 资源 ? 用户可以直接通过 podname 进行解析.

- 为什么需要监听 endpoints 资源 ? 如果 service 有配置 clusterNone, 也就是 headless 类型, 则需要使用到 endpoints 的地址.

- 为什么需要监听 namespace 资源 ? 需要判断请求域名的 namespace 段是否可用.

```go
// newdnsController creates a controller for CoreDNS.
func newdnsController(ctx context.Context, kubeClient kubernetes.Interface, opts dnsControlOpts) *dnsControl {
	// 自定义 dns controller 控制器对象
	dns := dnsControl{
		client:            kubeClient,
		selector:          opts.selector,
		namespaceSelector: opts.namespaceSelector,
		zones:             opts.zones,
	}

	// 实例化 service 资源的 informer 对象, 自定义 list 和 watch 的筛选方法.
	dns.svcLister, dns.svcController = object.NewIndexerInformer(
		&cache.ListWatch{
			ListFunc:  serviceListFunc(ctx, dns.client, api.NamespaceAll, dns.selector),
			WatchFunc: serviceWatchFunc(ctx, dns.client, api.NamespaceAll, dns.selector),
		},
		// 指定 service 资源类型
		&api.Service{},
		// 向 informer 注册 eventHandler 回调方法 
		cache.ResourceEventHandlerFuncs{AddFunc: dns.Add, UpdateFunc: dns.Update, DeleteFunc: dns.Delete,
		},
		// 注册 cache 的 index 方法
		cache.Indexers{svcNameNamespaceIndex: svcNameNamespaceIndexFunc, svcIPIndex: svcIPIndexFunc, svcExtIPIndex: svcExtIPIndexFunc},
		object.DefaultProcessor(object.ToService, nil),
	)

	// 实例化 pod infromer, 默认不开启
	if opts.initPodCache {
		dns.podLister, dns.podController = object.NewIndexerInformer(
			&cache.ListWatch{
				ListFunc:  podListFunc(ctx, dns.client, api.NamespaceAll, dns.selector),
				WatchFunc: podWatchFunc(ctx, dns.client, api.NamespaceAll, dns.selector),
			},
			&api.Pod{},
			cache.ResourceEventHandlerFuncs{AddFunc: dns.Add, UpdateFunc: dns.Update, DeleteFunc: dns.Delete},
			cache.Indexers{podIPIndex: podIPIndexFunc},
			object.DefaultProcessor(object.ToPod, nil),
		)
	}

	// 实例化 endpoints infromer
	if opts.initEndpointsCache {
		dns.epLock.Lock()
		dns.epLister, dns.epController = object.NewIndexerInformer(
			...
			// 定义 endpoints 资源
			&discovery.EndpointSlice{},
			cache.ResourceEventHandlerFuncs{AddFunc: dns.Add, UpdateFunc: dns.Update, DeleteFunc: dns.Delete},
			...
		)
		dns.epLock.Unlock()
	}

	// 实例化 namespace infromer
	dns.nsLister, dns.nsController = object.NewIndexerInformer(
		...
		&api.Namespace{},
		...
	)

	return &dns
}
```

#### informer evnetHandler

上面的那几个资源的 informer 注册的 eventHandler 都是一样的. add, delete, update 最后要么调用 updateModified, 要么 updateExtModified, 总之这两个方法都只是用 atomic 原子修改时间戳, 只有 service 绑定 externalIPs 时奇特.

可以发现 coredns dns controller 跟社区中其他的 controller 逻辑有些不一样. 像 kubernetes controller manager 中的那些控制器, 在 informer 注册的方法是把拿到的事件拆解后放到本地队列 workqueue 里, 然后启动几个 worker 循环调用 syncHandler 做状态和配置的同步.

而 coredns dns controller 只是更新时间戳, 这个时间戳是用来做 Dns Soa Serial number 的. dns server 每次更新记录后都需要变更下 serial number 序号, 这样一些 dns cahce 和 dns 从节点就可以感知是否有配置变更.

代码位置: `plugin/kubernetes/controller.go`

```go
func (dns *dnsControl) Add(obj interface{})               { dns.updateModified() }
func (dns *dnsControl) Delete(obj interface{})            { dns.updateModified() }
func (dns *dnsControl) Update(oldObj, newObj interface{}) { dns.detectChanges(oldObj, newObj) }

func (dns *dnsControl) detectChanges(oldObj, newObj interface{}) {
	obj := newObj
	if obj == nil {
		obj = oldObj
	}
	switch ob := obj.(type) {
	case *object.Service:
		// 获取需要更新哪些时间戳
		imod, emod := serviceModified(oldObj, newObj)
		if imod {
			// 更新时间戳
			dns.updateModified()
		}
		// 当 service 含有 externalIPs 时, 修改 extModified 时间戳.
		if emod {
			// 更新 ext 时间戳
			dns.updateExtModifed()
		}
	case *object.Pod:
		// 更新时间戳
		dns.updateModified()
	case *object.Endpoints:
		// 只有 endpoints 地址变更时才更新时间戳
		if !endpointsEquivalent(oldObj.(*object.Endpoints), newObj.(*object.Endpoints)) {
			// 更新时间戳
			dns.updateModified()
		}
	default:
		...
	}
}

// 把当前时间戳更新到 modified 里.
func (dns *dnsControl) updateModified() {
	unix := time.Now().Unix()
	atomic.StoreInt64(&dns.modified, unix)
}

// 同上
func (dns *dnsControl) updateExtModifed() {
	unix := time.Now().Unix()
	atomic.StoreInt64(&dns.extModified, unix)
}
```

informer 内部是有 store 缓存的, 他通过 list/watch 获取全量及增量的数据, 中间不仅会回调用户注册的 eventHandler, 而且维护本地 store 缓存和 indexer 索引. 

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301021144090.png)

#### 从 informer store 里检索 dns 记录

coredns server 收到 dns query 请求后, 从哪里获取域名对应的记录 ? 

其实就是 dnscontroller 里各个资源的 cache.Indexer 获取. 再详细的说, 从 indexer 和 store 里获取关联的对象, 经过数据组装后返回给 dns 客户端.

#### 启动控制器

`Run()` 启动上面实例化好的各个资源的 informer.

```go
// Run starts the controller.
func (dns *dnsControl) Run() {
	go dns.svcController.Run(dns.stopCh)
	if dns.epController != nil {
		go func() {
			dns.epLock.RLock()
			dns.epController.Run(dns.stopCh)
			dns.epLock.RUnlock()
		}()
	}
	if dns.podController != nil {
		go dns.podController.Run(dns.stopCh)
	}
	go dns.nsController.Run(dns.stopCh)
	<-dns.stopCh
}
```

### coredns k8s 插件如何处理 dns 请求 ?

coredns 每个插件都要实现 Plugin Handler 接口, 接口中定义了两个方法. `Name()` 返回插件名, `ServeDNS()` 用来处理域名查询. coredns 在初始化阶段会把所有插件的注册都插件链条里.

```go
type (
    Handler interface {
        ServeDNS(context.Context, dns.ResponseWriter, *dns.Msg) (int, error)
        Name() string
    }
)
```

coredns kubernetes 插件自然也实现了 `ServeDNS` 方法, 该逻辑通过请求的 Qtype 查询类型, 调用不同方法来解析域名.

```go
// ServeDNS implements the plugin.Handler interface.
func (k Kubernetes) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) (int, error) {
	state := request.Request{W: w, Req: r}

	...

	switch state.QType() {
	case dns.TypeA:
		records, truncated, err = plugin.A(ctx, &k, zone, state, nil, plugin.Options{})
	case dns.TypeAAAA:
		records, truncated, err = plugin.AAAA(ctx, &k, zone, state, nil, plugin.Options{})
	case dns.TypeTXT:
		records, truncated, err = plugin.TXT(ctx, &k, zone, state, nil, plugin.Options{})
	case dns.TypeCNAME:
		records, err = plugin.CNAME(ctx, &k, zone, state, plugin.Options{})
	case dns.TypePTR:
		records, err = plugin.PTR(ctx, &k, zone, state, plugin.Options{})
	case dns.TypeMX:
		records, extra, err = plugin.MX(ctx, &k, zone, state, plugin.Options{})
	case dns.TypeSRV:
		records, extra, err = plugin.SRV(ctx, &k, zone, state, plugin.Options{})
	case dns.TypeSOA:
		records, err = plugin.SOA(ctx, &k, zone, state, plugin.Options{})
	case dns.TypeAXFR, dns.TypeIXFR:
		return dns.RcodeRefused, nil
	case dns.TypeNS:
		records, extra, err = plugin.NS(ctx, &k, zone, state, plugin.Options{})
	default:
		...
		_, _, err = plugin.A(ctx, &k, zone, fake, nil, plugin.Options{})
	}

	if err != nil {
		return dns.RcodeServerFailure, err
	}
	...

	# 返回写入 dns reply
	m := new(dns.Msg)
	m.SetReply(r)
	m.Truncated = truncated
	m.Authoritative = true
	m.Answer = append(m.Answer, records...)
	m.Extra = append(m.Extra, extra...)
	w.WriteMsg(m)
	return dns.RcodeSuccess, nil
}
```

上面的 plugin.A, plugin.Txt, plugin.CNAME 都会落到 `Services()` 方法里. 接着定位到 `Records()` 方法.

代码位置: `plugin/kubernetes/kubernetes.go`

```go
// Services implements the ServiceBackend interface.
func (k *Kubernetes) Services(ctx context.Context, state request.Request, exact bool, opt plugin.Options) (svcs []msg.Service, err error) {
	switch state.QType() {
	case dns.TypeTXT:
		...

	case dns.TypeNS:
		...
	}

	# 处理默认的 ns 的 域名
	if isDefaultNS(state.Name(), state.Zone) {
		...
	}

	# 处理域名解析
	s, e := k.Records(ctx, state, false)

	# 过滤掉 CNAME 类型
	internal := []msg.Service{}
	for _, svc := range s {
		if t, _ := svc.HostType(); t != dns.TypeCNAME {
			internal = append(internal, svc)
		}
	}

	return internal, e
}
```

#### 判断是 service 还是 pod 请求

`Records()` 会从域名的特征判断请求域名是 pod 还是 svc 字符串. pod 走 `findPods` 查询方法, 其他情况都走 `findServices` 方法.

```go
// Records looks up services in kubernetes.
func (k *Kubernetes) Records(ctx context.Context, state request.Request, exact bool) ([]msg.Service, error) {
	r, e := parseRequest(state.Name(), state.Zone)
	if e != nil {
		return nil, e
	}

	// 给 namespace 是否合法
	if !k.namespaceExposed(r.namespace) {
		return nil, errNsNotExposed
	}

	// 请求的域名有 pod 特征, 则使用 findPods 来处理 
	if r.podOrSvc == Pod {
		pods, err := k.findPods(r, state.Zone)
		return pods, err
	}

	// 其他情况, 则使用 findServices 来处理 
	services, err := k.findServices(r, state.Zone)
	return services, err
}
```

#### 查询 service 对应的地址

`findServices()` 从 informer 的缓存中获取服务名对应的 service 列表, 然后进行遍历处理. 当 service 类型为 ServiceTypeExternalName 时, 返回 ExternalName 值做 CNAME 处理.

当 service 类型为 Headless 时, 且请求的 endpoint 不为空, 则把 service 对应的 endpoints 地址都返回. 剩下的情况就是 server 有配置 ClusterIP 集群地址, 那么就返回关联的 clusterIP 地址.

虽然在 `struct msg.Service` 结构体里定义了 Port 端口号, 这个是给其他调用方使用的, coredns 返回的 dns 记录是不含有 port 等属性信息的.

```go
func (k *Kubernetes) findServices(r recordRequest, zone string) (services []msg.Service, err error) {
	// service 为空则直接跳出
	if r.service == "" {
		if k.namespaceExposed(r.namespace) {
			// NODATA
			return nil, nil
		}
		return nil, errNoItems
	}
	// 默认为 error 为无记录
	err = errNoItems

	// 拼凑域名格式为 servicename.namespace
	idx := object.ServiceKey(r.service, r.namespace)

	// 通过 dnsController 的 svcLister.ByIndex 从缓存中获取 serviceList 列表
	serviceList = k.APIConn.SvcIndex(idx)

	// 转换下格式, 比如 service.staging.xioarui.local. 转成 /xiaorui/local/skydns/staging/service .
	zonePath := msg.Path(zone, coredns)
	for _, svc := range serviceList {
		// 如何 namespace 不一致, 且service不一致, 则忽略.
		if !(match(r.namespace, svc.Namespace) && match(r.service, svc.Name)) {
			continue
		}

		...

		// 如果 service 有绑定 ExternalName, 则需要返回 CNAME 记录.
		if svc.Type == api.ServiceTypeExternalName {
			if r.endpoint != "" {
				continue
			}
			s := msg.Service{Key: strings.Join([]string{zonePath, Svc, svc.Namespace, svc.Name}, "/"), Host: svc.ExternalName, TTL: k.ttl}
			if t, _ := s.HostType(); t == dns.TypeCNAME {
				...
				services = append(services, s)

				err = nil
			}
			continue
		}

		// 如果 service 是 headless 类型, 则返回 service endoptins 地址集.
		if svc.Headless() || r.endpoint != "" {
			if endpointsList == nil {
				endpointsList = endpointsListFunc()
			}

			for _, ep := range endpointsList {
				for _, eps := range ep.Subsets {
					for _, addr := range eps.Addresses {
						for _, p := range eps.Ports {
							...
							s := msg.Service{Host: addr.IP, Port: int(p.Port), TTL: k.ttl}
							...

							services = append(services, s)
						}
					}
				}
			}
			continue
		}

		// 如果 service 类型是 ClusterIP, 则返回 clusterIP 这个 vip.
		for _, p := range svc.Ports {
			...
			err = nil

			for _, ip := range svc.ClusterIPs {
				s := msg.Service{Host: ip, Port: int(p.Port), TTL: k.ttl}
				services = append(services, s)
			}
		}
	}
	return services, err
}
```

#### 查询 pods 对应的地址

`findPods()` 是处理 podname 的域名解析的方法, 其内部调用 `dnsController` 里的 `podLister.ByIndex` 来获取 pod, podLister 内部维护了 indexer 索引和 store 存储, 查询条件是格式化过的 podname.

> podname 大概是这样的 `nginx-8d52d677619-bp7br.nginx.default.svc.cluster.local`. 

```go
func (k *Kubernetes) findPods(r recordRequest, zone string) (pods []msg.Service, err error) {
	...

	// 判断 namespace 是否合法
	if !k.namespaceExposed(namespace) {
		return nil, errNoItems
	}

	// 这里的服务名其实就是 podname
	podname := r.service

	zonePath := msg.Path(zone, coredns)

	// 真奇葩, 居然用 ip 这个变量名.
	ip := ""

	// 把 podname 中的 `-` 符号映射到 `.` 或者 `:` 符号.
	if strings.Count(podname, "-") == 3 && !strings.Contains(podname, "--") {
		ip = strings.ReplaceAll(podname, "-", ".")
	} else {
		ip = strings.ReplaceAll(podname, "-", ":")
	}

	...

	err = errNoItems

	// 调用 dnscontroller 内的 `podLister.ByIndex` 从缓存中检索匹配的 pod 对象.
	for _, p := range k.APIConn.PodIndex(ip) {
		if ip == p.PodIP && match(namespace, p.Namespace) {
			s := msg.Service{Key: strings.Join([]string{zonePath, Pod, namespace, podname}, "/"), Host: ip, TTL: k.ttl}
			pods = append(pods, s)

			err = nil
		}
	}
	return pods, err
}
```

### coredns 返回 dns 记录的过程

`ServeDNS` 内部会调用 plugin 抽象化的各个记录类别的通用查询接口. plugin.XXX 方法会把 msg.Service 结构体转成真正的 dns 记录结构体, 然后把数据写到 writer 里. 这里的 writer 不是真正的 dns client, 而是附带 buffer 缓冲的自定义 writer. 只有等插件的调用链都执行完毕后, coredns 才会把 buffer 写到 conn 里.

代码位置: `plugin/backend_lookup.go`

```go
func (k Kubernetes) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) (int, error) {
	...

	switch state.QType() {
	case dns.TypeA:
		// Kubernetes 实现了 ServiceBackend interface.
		records, truncated, err = plugin.A(ctx, &k, zone, state, nil, plugin.Options{})
	case dns.TypeCNAME:
		records, err = plugin.CNAME(ctx, &k, zone, state, plugin.Options{})
		...
	default:
		忽略其他的类型
		...
	}

	...

	m := new(dns.Msg)
	m.SetReply(r)
	m.Answer = append(m.Answer, records...)
	...

	// 把返回值写入到 writer buffer 里.
	w.WriteMsg(m)
	return dns.RcodeSuccess, nil
}

func A(ctx context.Context, b ServiceBackend, zone string, state request.Request, previousRecords []dns.RR, opt Options) (records []dns.RR, truncated bool, err error) {
	// checkForApex 会调用 k8s 插件的 的 Services() 方法, 拿到的是 []msg.Service 记录.
	services, err := checkForApex(ctx, b, zone, state, opt)
	if err != nil {
		return nil, false, err
	}

	dup := make(map[string]struct{})

	for _, serv := range services {
		// what 为 dns type, ip 就是 ip 地址.
		what, ip := serv.HostType()

		switch what {
		case dns.TypeCNAME:
			...
			continue

		case dns.TypeA:
			// 使用 dup 表过滤掉相同的 host, 这里的 serv.Host 说的是 ip 地址.
			if _, ok := dup[serv.Host]; !ok {
				dup[serv.Host] = struct{}{}

				// 返回类型为 A 记录类型, 且值为 ip 地址.
				records = append(records, serv.NewA(state.QName(), ip))
			}

		case dns.TypeAAAA:
			...
		}
	}
	return records, truncated, nil
}

// 创建 A 记录的结构体
func (s *Service) NewA(name string, ip net.IP) *dns.A {
	return &dns.A{Hdr: dns.RR_Header{Name: name, Rrtype: dns.TypeA, Class: dns.ClassINET, Ttl: s.TTL}, A: ip}
}
```

### coredns query performance

coredns 官方给出的性能报告是 3w/s, k8s coredns deployment 配置里默认是有开启 cache 缓存插件的, cache 的 expire 是 30 秒. 

笔者也亲自测试过 dns query 性能, 在高配的服务器下也就 4w 左右. 在关闭 log 后, qps 是可以到 6w/s. 该性能相比社区中 dns server 差点意思. 😅

虽然 coredns 性能差点, 但好在通常大家对 dns server 的 qps 不是很敏感. 毕竟大部分业务代码不会频繁的使用短连接, 另外 linux 主机本身可以使用 nscd 做缓存, 一些应用框架也有对 dns 缓存功能.

| Query Type  | QPS   | Avg Latency (ms) | Memory delta (MB)    |
|-------------|-------|------------------|----------------------|
| external    | 31428 | 2.605            | +5                   | 
| internal    | 33918 | 2.62             | +5                   |

coredns 有哪些优化方法 ?

#### 合理配置 dns ndots 配置

默认情况下 k8s 域名解析需要来回折腾好几次才能解析到. 因为 pod 内部的 ndots 为 5.

`ndots:5` 是什么意思? 如果域名中 . 的数量小于 5, 就依次遍历 search 中的后缀并拼接上进行 DNS 查询.

```bash
# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.254.0.10
options ndots:5
```

**.** 小于 5 个, 就依次拼凑 search 后缀去 dns server 访问. **.** 大于等于 5 个, 则直接进行查询.

例如: 在某个 pod 中查询 xiaorui.default.svc.cluster.local 这个 service, 过程如下:

1. 需要请求的域名中有4个 **.**, 还是小于5, 所以尝试拼接上第一个search进行查询，即 xiaorui.default.svc.cluster.local.default.svc.cluster.local, 查不到该域名.
2. 继续尝试 xiaorui.default.svc.cluster.local.svc.cluster.local，查不到该域名.
3. 继续尝试 xiaorui.default.svc.cluster.local.cluster.local，仍然查不到该域名.
4. 尝试不加后缀，即 xiaorui.default.svc.cluster.local，查询成功，返回响应的 ClusterIP.

可以看到一个简单的 service 域名解析需要经过4轮解析才能成功，集群中充斥着大量无用的 DNS 请求.

建议在 Pod 的配置中把 ndots 调整到 2, 当访问本 namespace 的 service 时, 直接使用 service name 查询, 由于不满足 ndots 策略, 则使用 `xiaorui.default.svc.cluster.local` 访问, 该域名是合理的. 当访问 myns 命名空间的 xiaorui 服务时, 业务层如果使用 xiaorui.myns 请求时, 由于不满足 ndots 策略, 还是会追加 search 后缀, 需要折腾两次.

总结, 合理的方法是在业务端写全域名.

#### 使用 NodeLocal Dns Cache

NodeLocal DNSCache 通过在集群节点上作为 DaemonSet 运行 dns 缓存代理来提高集群 DNS 性能. 拦截 Pod 的 DNS 查询的请求, 外部域名请求不再请求中心 Coredns. 

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301021935175.png)

### 总结

值得一说的是 coredns 众多功能都是使用 plugin 插件来实现的, 包括 cache 和 log 等功能. kubernetes coredns controller 理所当然也是通过 plugin 实现的.

kubernetes coredns plugin 的源码和原理, 相比其他 k8s 组件来说好理解的多. 

**简化流程如下:**

- coredns 启动时初始化 kubernetes 插件, 并装载注册插件.
- 插件 kubernetes 中有个 dnscontroller 控制器, 它会实例化 service/pods/endpoints/ns 等资源的 informer 并发起监听. 当这些资源发生变更时, 会修改 informer 关联的 indexer 索引存储对象. 
- 当 coredns 有查询请求到来时, 尝试从 indexer 获取对象, 然后返回组装的 dns 数据.
  - 域名为 pod 特征
    - 从 podIndexer 中获取 podIP.
  - 其他情况
    - 当查询的 service 为 externalName 时, 则返回 CNAME 记录.
    - 当查询的 service 为 ClusterNone or headless 时, 则返回 service 对应的 endpoints 地址.
    - 当查询的 service 为 ClusterIP 时, 则返回 service 的 clusterIP.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301022021998.png)