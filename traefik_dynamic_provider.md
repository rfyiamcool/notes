## traefik 设计实现之配置的动态更新

### 实现简要

像 nginx 是支持配置更新，只是不支持动态的配置发现更新，需要外加插件才可以，不加插件只能手动改文件，然后通过 `nginx -s reload` 来更新，reload 的实现错误是 master 收到信号后重载配置，fork 新的 worker 子进程，最后优雅干掉老配置的 worker 子进程. 

> 基于 openresty 的 kong 和 apisix 都是支持动态配置更新的.

traefik 是支持配置的动态更新，那么 traefik 作为高性能网关，如何在不影响业务的情况下，实现配置的动态更新 ?

![traefik 源码分析](https://xiaorui.cc/wp-content/uploads/2022/11/Jietu20221123-124659.jpg)

通过分析代码后，得知动态更新主要依赖 watcher，listener, provider, switcher 设计来实现的，缺一不可呀. 😁

- watcher 动态配置的主逻辑，关联了 listener, provider.
- listener 实现具体配置在 traefik 里的更新.
- provider 实现发现配置及监听是否变更.
- switcher 抽象了 http.Handler, tcp.Handler 等逻辑，每次读取或变更时，都需要加锁以保证线程安全.

### 源码分析

#### 初始化阶段

首先 traefik 在 cmd 初始化阶段, 通过 `setupServer` 进行实例化 provider 和 watcher 对象.

```go
func setupServer() {
	...

	// 根据 provider 类型做实例化 provider.
	providerAggregator := aggregator.NewProviderAggregator(*staticConfiguration.Providers)

  // 创建 watcher 监听和更新配置.
	watcher := server.NewConfigurationWatcher(
		routinesPool,
		providerAggregator,
		getDefaultsEntrypoints(staticConfiguration),
		"internal",
	)
	...
}
```

#### watcher

watcher 实现了配置监听和更新的主逻辑，内会启动了三个协程.

\- `receiveConfigurations`, 监听配置更新, 对比本地配置并发送通知

\- `applyConfigurations`, 应用配置更新, 调用上层传递 listener 回调方法，使各个组件都被动更新

\- `startProviderAggregator`, 调用 provider 的 provide 接口，传递一个 chan 和 协程池wg

```go
// NewConfigurationWatcher creates a new ConfigurationWatcher.
func NewConfigurationWatcher(
	routinesPool *safe.Pool,
	pvd provider.Provider,
	defaultEntryPoints []string,
	requiredProvider string,
) *ConfigurationWatcher {
	return &ConfigurationWatcher{
	}
}

// Start the configuration watcher.
func (c *ConfigurationWatcher) Start() {
	c.routinesPool.GoCtx(c.receiveConfigurations)
	c.routinesPool.GoCtx(c.applyConfigurations)
	c.startProviderAggregator()
}

// Stop the configuration watcher.
func (c *ConfigurationWatcher) Stop() {
	close(c.allProvidersConfigs)
	close(c.newConfigs)
}

// AddListener adds a new listener function used when new configuration is provided.
func (c *ConfigurationWatcher) AddListener(listener func(dynamic.Configuration)) {
	if c.configurationListeners == nil {
		c.configurationListeners = make([]func(dynamic.Configuration), 0)
	}
	c.configurationListeners = append(c.configurationListeners, listener)
}
```

#### switcher

switcher 这个名字起的有点怪，traefik 各个组件都有 switcher 的实现，比如 http 里的 switcher 实现了 http.handler 接口，tcp 的 switcher 实现了 tcp.handler 接口.  switcher 本质就是 `sync.Map + interface{}` 组合，目的为了线程安全获取和动态更新 handler. 

这样当每次请求需要处理时，都会从 switcher.safe 里加锁获取 http.Handler, 当 watcher 配置动态更新的时候，也是加锁的更新 http.Handler.

下面是 http switcher 的结构设计:

```go
type HTTPHandlerSwitcher struct {
	handler *safe.Safe
}

func NewHandlerSwitcher(newHandler http.Handler) (hs *HTTPHandlerSwitcher) {
	return &HTTPHandlerSwitcher{
		handler: safe.New(newHandler),
	}
}

func (h *HTTPHandlerSwitcher) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	handlerBackup := h.handler.Get().(http.Handler)
	handlerBackup.ServeHTTP(rw, req)
}

func (h *HTTPHandlerSwitcher) GetHandler() (newHandler http.Handler) {
	handler := h.handler.Get().(http.Handler)
	return handler
}

func (h *HTTPHandlerSwitcher) UpdateHandler(newHandler http.Handler) {
	h.handler.Set(newHandler)
}
```

#### listener

配置的动态更新是通过 `listener` 来实现的.  traefik 里有不少动态更新，这里重点说下两个配置更新.

- server trasnports 连接池
- routers 路由匹配策略

##### server transports 连接池的更新

server transports 的监听器在当配置更新时，会对 `RoundTripper`  管理器加锁, 然后新建 transport 连接池, 老的池子不去管，因为 http 组件也是每次通过 Get()  实时获取的 roundTripper，在当前的请求都处理完毕后，http 就不会再复用引用这个要销毁的 `RoundTripper`,  后面依赖空闲超时 `idleConnTimeout` 去关闭连接. 

```go
// Server Transports
watcher.AddListener(func(conf dynamic.Configuration) {
	roundTripperManager.Update(conf.HTTP.ServersTransports)
})
```

roundTripper update 的逻辑实现，先去除老的配置，并分析配置否是有变更，对于产生变更的新建连接池，对于新增的 transport 配置则直接新创建.

```go
func (r *RoundTripperManager) Update(newConfigs map[string]*dynamic.ServersTransport) {
	r.rtLock.Lock()
	defer r.rtLock.Unlock()

	for configName, config := range r.configs {
		// 去除老的配置
		newConfig, ok := newConfigs[configName]
		if !ok {
			delete(r.configs, configName)
			delete(r.roundTrippers, configName)
			continue
		}

		// 如果新旧配置相等, 则无需更新
		if reflect.DeepEqual(newConfig, config) {
			continue
		}

		var err error
		// 对于有变更的配置，实例化新 roundTripper 连接池.
		r.roundTrippers[configName], err = r.createRoundTripper(newConfig)
	}

	for newConfigName, newConfig := range newConfigs {
		if _, ok := r.configs[newConfigName]; ok {
			continue
		}

		var err error
		// 对于没有的，实例化连接池
		r.roundTrippers[newConfigName], err = r.createRoundTripper(newConfig)
	}

	r.configs = newConfigs
}
```

##### routers 路由规则的更新

关于 router 的监听器，当配置更新时，重新创建 tcpRouters 和 udpRouters 路由，然后分别在 tcp 和 udp 的 entrypoint 上做更新, 遍历更新到各个组件的 switcher 上，本质就是个加了锁的 value.

```go
// Switch router
watcher.AddListener(switchRouter(routerFactory, serverEntryPointsTCP, serverEntryPointsUDP))
```

在 tcp 和 udp 上更新 routers.

```go
func switchRouter(routerFactory *server.RouterFactory, serverEntryPointsTCP server.TCPEntryPoints, serverEntryPointsUDP server.UDPEntryPoints) func(conf dynamic.Configuration) {
	return func(conf dynamic.Configuration) {
		rtConf := runtime.NewConfig(conf)

		routers, udpRouters := routerFactory.CreateRouters(rtConf)

		serverEntryPointsTCP.Switch(routers)
		serverEntryPointsUDP.Switch(udpRouters)
	}
}

// Switch the TCP routers.
func (eps TCPEntryPoints) Switch(routersTCP map[string]*tcprouter.Router) {
	for entryPointName, rt := range routersTCP {
		eps[entryPointName].SwitchRouter(rt)
	}
}

// SwitchRouter switches the TCP router handler.
func (e *TCPEntryPoint) SwitchRouter(rt *tcprouter.Router) {
    ...
	httpHandler := rt.GetHTTPHandler()
    ...
	e.httpServer.Switcher.UpdateHandler(httpHandler)
	rt.SetHTTPSForwarder(e.httpsServer.Forwarder)
    ...
	httpsHandler := rt.GetHTTPSHandler()
	e.httpsServer.Switcher.UpdateHandler(httpsHandler)
	e.switcher.Switch(rt)
	e.http3Server.Switch(rt)
}
```

#### provider

provider 是用来实现监听配置的动态更新以及获取变更通知的。逻辑相对简单，就是监听配置是否有变更，当发生变更后，通知给 watcher， watcher 在把事件传递给一个个的 listener 做具体的配置变更.

比如可以把配置的放到 etcd 上，当在 etcd 上做了更新后，通知给 traefik 做变更. traefik 实现了不少  provider，有本地文件、docker、etcd、redis、consul、http、 k8s crd 等.

##### file provider 实现

基于文件的 provider 的实现，源码还是简单的. 首先需要实现 Provide 接口，该方法通过 inotify 来监听文件的更新事件，当发生变动时，会通知给上层传递的 `configurationChan chan<- dynamic.Message` 通道. 

```go
func (p *Provider) Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error {
	configuration, err := p.BuildConfiguration()
	if err != nil {
		return err
	}

	if p.Watch {
		var watchItem string
		...
		if err := p.addWatcher(pool, watchItem, configurationChan, p.watcherCallback); err != nil {
			return err
		}
	}

	sendConfigToChannel(configurationChan, configuration)
	return nil
}

func (p *Provider) watcherCallback(configurationChan chan<- dynamic.Message, event fsnotify.Event) {
	sendConfigToChannel(configurationChan, configuration)
}

func sendConfigToChannel(configurationChan chan<- dynamic.Message, configuration *dynamic.Configuration) {
	configurationChan <- dynamic.Message{
		ProviderName:  "file",
		Configuration: configuration,
	}
}
```

##### kv provider 实现

traefik 使用 `github.com/kvtools/valkeyrie` 实现常见 kv 存储的读写接口, 所以在 traefik 看不到各类 kv 数据库读写实现.  kv provider 的实现跟 file provider 大同小异. 

```go
// Provide allows the docker provider to provide configurations to traefik using the given configuration channel.
func (p *Provider) Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error {
	configuration, err := p.buildConfiguration(ctx)
	if err != nil {
	} else {
		configurationChan <- dynamic.Message{
			ProviderName:  p.name,
			Configuration: configuration,
		}
	}

	pool.GoCtx(func(ctxPool context.Context) {
		err := p.watchKv(ctxLog, configurationChan)
	})

	return nil
}

func (p *Provider) watchKv(ctx context.Context, configurationChan chan<- dynamic.Message) error {
	operation := func() error {
		events, err := p.kvClient.WatchTree(ctx, p.RootKey, nil)
		for {
			select {
				...
			case _, ok := <-events:
				configuration, errC := p.buildConfiguration(ctx)
				if configuration != nil {
					configurationChan <- dynamic.Message{
						ProviderName:  p.name,
						Configuration: configuration,
					}
				}
			}
		}
	}
	...
	return nil
}
```

### 总结

traefik 的动态配置更新是由 watcher，listener, provider, switcher 组合实现的.

- watcher 动态配置的主逻辑，关联了 listener, provider.
- listener 实现具体配置在 traefik 里的更新.
- provider 实现发现配置及监听是否变更.
- switcher 抽象了 http.Handler, tcp.Handler 等逻辑，每次读取或变更时，都需要加锁以保证线程安全.

traefik 源码抽象的好，组合起来也顺，但分析源码时还是有些绕的，每个单元看起来都好懂，反而衔接起来反而不易懂。