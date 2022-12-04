## traefik 设计实现之 http、http2 和 grpc 代理

![https://doc.traefik.io/traefik/assets/img/traefik-architecture.png](https://doc.traefik.io/traefik/assets/img/traefik-architecture.png)

traefik 对于 http、http2、http3 的反向代理，没有专门去造轮子解决拆包及请求转发，而是引用 golang 标准库中 `httputil.ReveryProxy` 实现反向代理. 

另外观察了社区中其他 golang 的网关实现，也多使用 `httputil.ReveryProxy` 实现.

### 连接池管理器 RoundTripperManager

#### RoundTripperManager 作用

`RoundTripper` 可以理解为一个连接池管理器，在连接池里可以配置连接的读写超时，空闲超时，池子大小，缓冲大小等，一个连接池是可以管理多个后端主机. 像 golang net/http client 标准库，如不专门配置 transport, 那么就使用 http 库包中定义的默认 Transport.

代码位置 `net/http/transport.go`

```go
func (c *Client) transport() RoundTripper {
	if c.Transport != nil {
		return c.Transport
	}
	return DefaultTransport
}

var DefaultTransport RoundTripper = &Transport{
	DialContext: defaultTransportDialContext(&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}),
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```

那么 `RoundTripperManager` 顾名思义就是用来管理多个 `http.RoundTripper` 对象,  通过 `createRoundTripper()` 创建 http 和 http2 transport 连接池对象，`http.Transport` 不仅可以配置连接池，读写超时等常规配置，还可以配置 TLS 证书. 

另外 http.Transport 可以在 entrypoints 里配置，然后在 services 里使用对应的 entrypoints 的连接池，另外也可以配置专门的 `serversTransport` 配置项.

[https://doc.traefik.io/traefik/routing/services/#serverstransport_1](https://doc.traefik.io/traefik/routing/services/#serverstransport_1)

**连接池配置**

entrypoints 配置

```yaml
entryPoints:
  xiaorui-http:
    address: ":8888"
    transport:
      lifeCycle:
        requestAcceptGraceTimeout: 42
        graceTimeOut: 42
      respondingTimeouts:
        readTimeout: 42
        writeTimeout: 42
        idleTimeout: 42
```

serversTransports 配置

```yaml
http:
  serversTransports:
    mytransport:
      maxIdleConnsPerHost: 7
      certificates:
        - certFile: foo.crt
          keyFile: bar.crt
      forwardingTimeouts:
        dialTimeout: "1s"
        idleConnTimeout: "1s"
```



#### 分析 RoundTripperManager 主要代码:

**Get**

根据 name 获取 `http.RoundTripper` 连接池, traefik 会把 roundTripperManager 传递给 servcies 的代理层，当进行请求转发时，通过 Get() 获取对应的 roundTripper 进行代理转发.

**createRoundTripper**

根据配置创建 http.RoundTripper 对象

**Update**

更新连接池配置，这里可以跟 provider 动态配置发现关联，在启动阶段进行监听，当配置更新时进行 Update 更新配置，更新时为了避免影响当前的请求，所以直接加锁替换原来的连接池，旧的连接池当不再引用时，等待空闲超时来回收连接.

代码位置: `pkg/server/service/roundtripper.go`

```go
type RoundTripperManager struct {
	rtLock        sync.RWMutex
	roundTrippers map[string]http.RoundTripper
	configs       map[string]*dynamic.ServersTransport

	spiffeX509Source SpiffeX509Source
}

// 根据 name 获取 http.RundTriper 配置
func (r *RoundTripperManager) Get(name string) (http.RoundTripper, error) {
	r.rtLock.RLock()
	defer r.rtLock.RUnlock()

	if rt, ok := r.roundTrippers[name]; ok {
		return rt, nil
	}

	return nil, fmt.Errorf("servers transport not found %s", name)
}

// 根据配置创建 http.RoundTripper 对象
func (r *RoundTripperManager) createRoundTripper(cfg *dynamic.ServersTransport) (http.RoundTripper, error) {
	dialer := &net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}

	transport := &http.Transport{
		DialContext:           dialer.DialContext,
		MaxIdleConnsPerHost:   cfg.MaxIdleConnsPerHost,
		IdleConnTimeout:       90 * time.Second,
		TLSHandshakeTimeout:   10 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,
		ReadBufferSize:        64 * 1024,
		WriteBufferSize:       64 * 1024,
	}

	if cfg.InsecureSkipVerify || len(cfg.RootCAs) > 0 || len(cfg.ServerName) > 0 || len(cfg.Certificates) > 0 || cfg.PeerCertURI != "" {
		transport.TLSClientConfig = &tls.Config{
			ServerName:         cfg.ServerName,
			InsecureSkipVerify: cfg.InsecureSkipVerify,
			RootCAs:            createRootCACertPool(cfg.RootCAs),
			Certificates:       cfg.Certificates.GetCertificates(),
		}
		}
	}

	if cfg.DisableHTTP2 {
		return transport, nil
	}

	return newSmartRoundTripper(transport, cfg.ForwardingTimeouts)
}

// 更新连接池配置，这里可以跟 provider 动态配置发现关联，在启动阶段进行监听，当配置更新时进行 Update 更新配置，更新时为了避免影响当前的请求，所以直接加锁替换原来的连接池，旧的连接池当不再引用时，等待空闲超时来回收连接.
func (r *RoundTripperManager) Update(newConfigs map[string]*dynamic.ServersTransport) {
	r.rtLock.Lock()
	defer r.rtLock.Unlock()

	for configName, config := range r.configs {
		newConfig, ok := newConfigs[configName]
		if !ok {
			delete(r.configs, configName)
			delete(r.roundTrippers, configName)
			continue
		}

		if reflect.DeepEqual(newConfig, config) {
			continue
		}

		var err error
		r.roundTrippers[configName], err = r.createRoundTripper(newConfig)
		if err != nil {
			log.Error().Err(err).Msgf("Could not configure HTTP Transport %s, fallback on default transport", configName)
			r.roundTrippers[configName] = http.DefaultTransport
		}
	}

	for newConfigName, newConfig := range newConfigs {
		if _, ok := r.configs[newConfigName]; ok {
			continue
		}

		var err error
		r.roundTrippers[newConfigName], err = r.createRoundTripper(newConfig)
		if err != nil {
			log.Error().Err(err).Msgf("Could not configure HTTP Transport %s, fallback on default transport", newConfigName)
			r.roundTrippers[newConfigName] = http.DefaultTransport
		}
	}

	r.configs = newConfigs
}
```

####  那么 grpc 和 http2 是如何进行转发的 ?

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212041134793.png)

grpc 的网络协议也是 http2，http2 的 transport 是通过 `newSmartRoundTripper` 创建. 本质是 golang 扩展包 `golang.org/x/net/http2` 里有 http2 的支持.

代码位置: `pkg/server/service/smart_roundtripper.go`

```go
func newSmartRoundTripper(transport *http.Transport, forwardingTimeouts *dynamic.ForwardingTimeouts) (http.RoundTripper, error) {
	transportHTTP1 := transport.Clone()

	transportHTTP2, err := http2.ConfigureTransports(transport)
	if err != nil {
		return nil, err
	}

	transportH2C := &h2cTransportWrapper{
		Transport: &http2.Transport{
			DialTLS: func(network, addr string, cfg *tls.Config) (net.Conn, error) {
				return net.Dial(network, addr)
			},
			AllowHTTP: true,
		},
	}

	if forwardingTimeouts != nil {
		transportH2C.ReadIdleTimeout = time.Duration(forwardingTimeouts.ReadIdleTimeout)
		transportH2C.PingTimeout = time.Duration(forwardingTimeouts.PingTimeout)
	}

	transport.RegisterProtocol("h2c", transportH2C)

	return &smartRoundTripper{
		http2: transport,
		http:  transportHTTP1,
	}, nil
}
```

### HTTP 和 HTTP2代理

为 services 的主机列表遍历实例化反向代理对象，并生成负载均衡器, 值得注意的是，不管是 loadbalancer 和 reverseproxy 都是 http.Handler 接口，而且 router 也实现了对应的 http.Handler 接口，这样关联到 entryPoints 入口层，会先调用 routers 的 rules ，继而调用 servcies 对应的 loadbalancer，最后根据调度算法找到对应的 proxy 对象完成请求转发.

<img src="https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212031722503.png" style="zoom: 33%;" />

#### Servcies Manager 管理器

实例化 Manager 对象时，需传入 servcies 配置和 roundTripper 管理器，由 routers 层调用其 `BuildHTTP` 来进行初始化 service 对应的 loadBalaner 均衡器, 这里 `BuildHTTP()` 返回的对象是 http.Handler. 

`pkg/server/service/service.go`

```go
func NewManager(configs map[string]*runtime.ServiceInfo, metricsRegistry metrics.Registry, routinePool *safe.Pool, roundTripperManager RoundTripperGetter) *Manager {
	return &Manager{
		routinePool:         routinePool,
		roundTripperManager: roundTripperManager,
		services:            make(map[string]http.Handler),
		configs:             configs,
		healthCheckers:      make(map[string]*healthcheck.ServiceHealthChecker),
	}
}

// BuildHTTP Creates a http.Handler for a service configuration.
func (m *Manager) BuildHTTP(rootCtx context.Context, serviceName string) (http.Handler, error) {
	serviceName = provider.GetQualifiedName(ctx, serviceName)
	handler, ok := m.services[serviceName]
	if ok {
		return handler, nil
	}

	conf, ok := m.configs[serviceName]
	if !ok {
		return nil, fmt.Errorf("the service %q does not exist", serviceName)
	}

	...

	var lb http.Handler

	switch {
	case conf.LoadBalancer != nil:
		lb, err = m.getLoadBalancerServiceHandler(ctx, serviceName, conf)
	case conf.Weighted != nil:
		lb, err = m.getWRRServiceHandler(ctx, serviceName, conf.Weighted)
	case conf.Mirroring != nil:
		lb, err = m.getMirrorServiceHandler(ctx, conf.Mirroring)
    ...
	}

	m.services[serviceName] = lb

	return lb, nil
}
```

#### 构建loadbalancer及反向代理

从 `RoundTripperManager`里获取对应的 `roundTripper` 连接池，遍历后端主机列表，并用原生库 `buildSingleHostProxy -> httputil.ReverseProxy` 对每个主机生成代理对象，最后使用这些代理对象构建负载均衡器. 不管是负载均衡器及内部的反向代理对象都实现 `http.Handler` 接口.

`pkg/server/service/service.go`

```go
func (m *Manager) getLoadBalancerServiceHandler(ctx context.Context, serviceName string, info *runtime.ServiceInfo) (http.Handler, error) {
	service := info.LoadBalancer

    // 设置 header
	passHostHeader := dynamic.DefaultPassHostHeader
	if service.PassHostHeader != nil {
		passHostHeader = *service.PassHostHeader
	}

    // 从 roundTripper 获取连接池
	roundTripper, err := m.roundTripperManager.Get(service.ServersTransport)
	if err != nil {
		return nil, err
	}

  // 实例化负载均衡器
	lb := wrr.New(service.Sticky, service.HealthCheck != nil)

  // 随机洗牌
	for _, server := range shuffle(service.Servers, m.rand) {
		target, err := url.Parse(server.URL)
		if err != nil {
			return nil, fmt.Errorf("error parsing server URL %s: %w", server.URL, err)
		}

    // 创建反向代理对象
		proxy := buildSingleHostProxy(target, passHostHeader, time.Duration(flushInterval), roundTripper, m.bufferPool)

    // 把对象加入到 lb 里
		lb.Add(proxyName, proxy, nil)
	}

	return lb, nil
}
```

借助 `httputil.ReverseProxy` 实现反向代理，只需传入转发目标的 url 和连接池就可以创建代理.  看似简单粗暴，深思又是大道至简. 以前写过 httputil.ReverseProxy 的源码解析，这里就不再复述了.

```go
func buildSingleHostProxy(target *url.URL, passHostHeader bool, flushInterval time.Duration, roundTripper http.RoundTripper, bufferPool httputil.BufferPool) http.Handler {
	return &httputil.ReverseProxy{
		Director:      directorBuilder(target, passHostHeader),
		Transport:     roundTripper,
		FlushInterval: flushInterval,
		BufferPool:    bufferPool,
		ErrorHandler:  errorHandler,
	}
}
```

### 健康检查

在构建 loadbalancer 的时候, 不仅会实例化后端主机的反向代理, 也会对该 lb 创建一个 healthChecker 健康检查器.

```go
type Manager struct {
	...
	healthCheckers map[string]*healthcheck.ServiceHealthChecker
}

// LaunchHealthCheck launches the health checks.
func (m *Manager) LaunchHealthCheck(ctx context.Context) {
	for serviceName, hc := range m.healthCheckers {
		logger := log.Ctx(ctx).With().Str(logs.ServiceName, serviceName).Logger()
		go hc.Launch(logger.WithContext(ctx))
	}
}
```

ServiceHealthChecker 代码位置: `pkg/healthcheck/healthcheck.go`

```go
func (shc *ServiceHealthChecker) Launch(ctx context.Context) {
	ticker := time.NewTicker(shc.interval) // 默认 30s
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return

		case <-ticker.C:
			// 遍历后端主机实例，当发现健康失败时, 更新 lb 状态.
			for proxyName, target := range shc.targets {
				up := true
				if err := shc.executeHealthCheck(ctx, shc.config, target); err != nil {
					up = false
				}

				shc.balancer.SetStatus(ctx, proxyName, up)

				statusStr := runtime.StatusDown
				if up {
					statusStr = runtime.StatusUp
				}
				shc.info.UpdateServerStatus(target.String(), statusStr)
			}
		}
	}
}
```

健康检查支持两种模式:

- http 模式，判断返回的 code 是否是 200
- grpc 模式，需要 grpc server 引入 health 包，这样 grpc.client 才可根据 resp.Status 子弹判断是否正常.

```go
func (shc *ServiceHealthChecker) executeHealthCheck(ctx context.Context, config *dynamic.ServerHealthCheck, target *url.URL) error {
	if config.Mode == modeGRPC {
		return shc.checkHealthGRPC(ctx, target)
	}
	return shc.checkHealthHTTP(ctx, target)
}
```

checkHealthHTTP 和 checkHealthGRPC 实现没什么好说的.