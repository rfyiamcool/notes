## 源码分析 kubernetes ingress nginx controller 控制器的实现原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301010021497.png)

本文基于 kubernetes/ingress-nginx `v1.5.1` 源码分析, ingress-nginx 里的 controller 控制器是 golang 开发的. 而 ingress-nginx 容器内使用了官方的 nginx, 没有直接使用 openresty, 原生 nginx 编译时打入了 lua / luajit 模块, 但引用的三方的库包是属于 openresty 社区里的. 在这看 ingress-nginx 详细的 lua 库包引用信息.

[https://github.com/kubernetes/ingress-nginx/blob/21aa7f55a3/images/nginx/rootfs/build.sh](https://github.com/kubernetes/ingress-nginx/blob/21aa7f55a3/images/nginx/rootfs/build.sh)

#### ingress-nginx 为什么没有直接使用 openresty?

在 github issue 列表中没有找到答案. 按理来说社区方面 openresty 要比 nginx 更加开放, 但毕竟没有 nginx 的背景和后台金主.

像社区的 kong 和 apache apisix 是基于 openresty 开发的, 另外他们也在社区中开源了自己的 ingress-controller.

#### ingress-nginx 项目地址:

[https://github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)

#### ingress-nginx 核心函数调用关系流程:

实现过程略显复杂, 故图中忽略细节.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301010007853.png)

### 实例化 nginx ingress controller 控制器

```go
// NewNGINXController creates a new NGINX Ingress controller.
func NewNGINXController(config *Configuration, mc metric.Collector) *NGINXController {
	// 读取 resulv.conf 文件，获取 dns server 地址集合
	h, err := dns.GetSystemNameServers()

	n := &NGINXController{
		stopCh:   make(chan struct{}),

		// informer 注册 eventHandler 会往这个 chan 发送事件.
		updateCh: channels.NewRingChannel(1024),
		Proxy: &tcpproxy.TCPProxy{},
		command: NewNginxCommand(),
	}

	// 实例化 store 对象, 可以把 store 想成一个有各种数据的缓存的存储, 内部也有 informer.
	n.store = store.New(
		...
		n.updateCh, // 把上面实例化的 updateCh 传进去了
		...
	)

	// 实例化 queue, 并且在 queue里注册了回调方法, syncIngress 是 nginx ingress controller 最核心的同步方法.
	n.syncQueue = task.NewTaskQueue(n.syncIngress)

	// 用在 inotify 文件监听的回调方法
	onTemplateChange := func() {
		// 从 `/etc/nginx/template/nginx.tmpl` 读取预设的模板, 然后进行解析生成 template 对象.
		template, err := ngx_template.NewTemplate(nginx.TemplatePath)
		if err != nil {
			klog.ErrorS(err, "Error loading new template")
			return
		}

		n.t = template

		// 向 queue 传递事件, 平滑热加载 nginx 配置
		n.syncQueue.EnqueueTask(task.GetDummyObject("template-change"))
	}

	// 从 `/etc/nginx/template/nginx.tmpl` 读取预设的模板, 然后进行解析生成 template 对象.
	ngxTpl, err := ngx_template.NewTemplate(nginx.TemplatePath)
	if err != nil {
		klog.Fatalf("Invalid NGINX configuration template: %v", err)
	}

	n.t = ngxTpl

	// 使用 inotify 机制异步监听 nginx.tmpl 模板文件, 当模板文件发生变更时, 则回调 onTemplateChange 方法, 重新读取模板并构建模板对象, 然后同步配置
	file.NewFileWatcher(nginx.TemplatePath, onTemplateChange)

	// 获取 geoip 目录下的相关文件, v4.4.2 里当前就只有三个文件, 分别是 geoip.dat, geoIPASNum.dat, geoLiteCity.dat 数据文件.
	filesToWatch := []string{}
	err = filepath.Walk("/etc/nginx/geoip/", func(path string, info os.FileInfo, err error) error {
		filesToWatch = append(filesToWatch, path)
		return nil
	})

	for _, f := range filesToWatch {
		// 异步监听 geoip dat 数据文件, 当发生增删改时, 重新平滑热加载 nginx 配置.
		_, err = file.NewFileWatcher(f, func() {
			n.syncQueue.EnqueueTask(task.GetDummyObject("file-change"))
		})
	}

	return n
}
```

### 服务启动入口

`Start()` 启动 nginx controller 控制器, 其原理如下:

1. 启动 `store.Run()`, 内部会启动 informer 监听并维护各资源的本地缓存 ;
2. 进行 leader election 选举, 只有主实例才可以执行状态同步的逻辑 ;
3. 启动 nginx 进程, 指定配置为 `/etc/nginx/nginx.conf` ;
4. 启动 syncQueue 里 run 方法, 该方法内部会从队列中读取任务, 并调用 syncIngress 来同步 nginx 配置 ;
5. 前面 nginx 启动使用时, 只是使用了默认的 nginx.conf, 里面几乎没什么东西, 这里通过主动通知 syncqueue, 然后 syncqueue worker 调度 syncIngress 来同步配置并完成热加载 ;
6. 启动一个协程每隔 5秒进行临时配置文件清理 ;
7. 从 informer 导入的 updateCh 获取任务, 并写到 syncQueue 队列里.

```go
func (n *NGINXController) Start() {
	klog.InfoS("Starting NGINX Ingress controller")

	// 启动 store, 内部会启动 informer 监听并维护各资源的本地缓存.
	n.store.Run(n.stopCh)

	// 进行选举, 只有主实例才可以执行状态同步的更新的逻辑.
	setupLeaderElection(&leaderElectionConfig{
		Client:     n.cfg.Client,
		ElectionID: electionID,
		OnStartedLeading: func(stopCh chan struct{}) {
			if n.syncStatus != nil {
				// 开启状态的同步更新
				go n.syncStatus.Run(stopCh)
			}
		},
	})

	// 配置进程组, 使用 cmd 创建的程序都所属相同的 pgid 组id, 这样杀掉进程时可以按照进程组杀, 避免有遗漏的进程.
	cmd := n.command.ExecCommand()
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Setpgid: true,
		Pgid:    0,
	}

	// 加载 ssl proxy
	if n.cfg.EnableSSLPassthrough {
		n.setupSSLProxy()
	}

	// 启动 nginx 进程, 指定配置为 `/etc/nginx/nginx.conf`.
	n.start(cmd)

	// 启动 syncQueue 里 run 方法, 该方法内部会从队列中读取任务, 并调用 syncIngress 来同步 nginx 配置.
	go n.syncQueue.Run(time.Second, n.stopCh)

	// 前面 nginx 启动使用时, 只是使用了默认的 nginx.conf, 里面几乎没什么东西, 这里通过主动传任务到 syncqueue, 然后 syncqueue 利用 syncIngress 来同步配置并完成热加载
	n.syncQueue.EnqueueTask(task.GetDummyObject("initial-sync"))

	go func() {
		for {
			time.Sleep(5 * time.Minute)

			// 启动一个异步 gc 协程, 清理 nginx 临时文件, 临时文件的名字前缀有 `nginx-cfg` 字符串, 当满足临时文件特征, 且更改超过5分钟则删除该临时文件 
			cleanTempNginxCfg()
		}
	}()

	for {
		select {
		case err := <-n.ngxErrCh:
			...
		case event := <-n.updateCh.Out():
			// 从 informer 拿到更改事件
			if evt, ok := event.(store.Event); ok {
				// 如果事件的类型是配置相关的, 则通知给 syncqueue, 让 syncqueue 的内部去同步配置.
				if evt.Type == store.ConfigurationEvent {
					n.syncQueue.EnqueueTask(task.GetDummyObject("configmap-change"))
					continue
				}

				// 同上, 只是任务可跳过.
				n.syncQueue.EnqueueSkippableTask(evt.Obj)
			}
		case <-n.stopCh:
			return
		}
	}
}
```

### 监听 informer 事件

`store` 实例化了 nginx ingress controller 所需资源的 informer 和 lister, 并注册 eventHandler 方法.

源码位置: [https://github.com/kubernetes/ingress-nginx/blob/main/internal/ingress/controller/store/store.go](https://github.com/kubernetes/ingress-nginx/blob/main/internal/ingress/controller/store/store.go)

```go
func New(
	namespace string,
	...
	updateCh *channels.RingChannel) Storer {

	store := &k8sStore{
		informers:             &Informer{},		// 各资源 informers 的集合
		listers:               &Lister{},		// informers listers 集合
		updateCh:              updateCh,		// 通知给 syncqueue 的通道
		backendConfig:         ngx_config.NewDefault(), // 获取 nginx 模板中需要的默认变量, 比如缓冲大小呀, keepalive 配置, http2 相关配置等, 另外默认 WorkerProcesses 为当前的cpu核心数. 
	}

	// 实例化 nginx ingress controller 所需资源的 informer 和 lister.
	store.informers.Ingress = infFactory.Networking().V1().Ingresses().Informer()
	store.listers.Ingress.Store = store.informers.Ingress.GetStore()
	store.informers.EndpointSlice = infFactory.Discovery().V1().EndpointSlices().Informer()
	store.listers.EndpointSlice.Store = store.informers.EndpointSlice.GetStore()
	store.informers.Secret = infFactorySecrets.Core().V1().Secrets().Informer()
	store.listers.Secret.Store = store.informers.Secret.GetStore()
	store.informers.ConfigMap = infFactoryConfigmaps.Core().V1().ConfigMaps().Informer()
	store.listers.ConfigMap.Store = store.informers.ConfigMap.GetStore()
	store.informers.Service = infFactory.Core().V1().Services().Informer()
	store.listers.Service.Store = store.informers.Service.GetStore()

	...
	// 巴拉巴拉, 实现了各类 informer 的 eventHandler 方法.
	...

	// 监听 ingress 资源并注册方法
	store.informers.Ingress.AddEventHandler(ingEventHandler)

	// 监听 endpoints 资源并注册方法
	store.informers.EndpointSlice.AddEventHandler(epsEventHandler)

	// 监听 secret 资源并注册方法
	store.informers.Secret.AddEventHandler(secrEventHandler)

	// 监听 configmap 资源并注册方法
	store.informers.ConfigMap.AddEventHandler(cmEventHandler)

	// 监听 service 资源并注册方法
	store.informers.Service.AddEventHandler(serviceHandler)

	return store
}
```

**ingress informer eventhandler**

下面是 store 里 ingress informer eventhandler 的实现, 收到事件后首先更新本地的缓存, 然后给 updateCh 传递通知.

```go
ingEventHandler := cache.ResourceEventHandlerFuncs{
	AddFunc: func(obj interface{}) {
		ing, _ := toIngress(obj)

		// 过滤不合法的 ns
		if !watchedNamespace(ing.Namespace) {
			return
		}

		// 更新 store 里 ingress 的缓存
		store.syncIngress(ing)
		
		// 更新 store 里 secret 的缓存
		store.syncSecrets(ing)

		// 向 updateCh 传递事件, 让 syncqueue 同步并加载配置
		updateCh.In() <- Event{
			Type: CreateEvent,
			Obj:  obj,
		}
	},
	DeleteFunc: ingDeleteHandler,
	UpdateFunc: func(old, cur interface{}) {
		oldIng, _ := toIngress(old)
		curIng, _ := toIngress(cur)

		if !watchedNamespace(oldIng.Namespace) {
			return
		}

		// 同步 store 缓存, 并传递更新事件
		store.syncIngress(curIng)
		store.updateSecretIngressMap(curIng)
		store.syncSecrets(curIng)

		updateCh.In() <- Event{
			Type: UpdateEvent,
			Obj:  cur,
		}
	},
}
```

**service informer eventHandler**

下面是 store 里 service informer eventHandler 的实现原理. 这里关注 service type 为 externalName 的, externalName 是个特殊的类型.

```go
	serviceHandler := cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			svc := obj.(*corev1.Service)
			// 只关注 service type 为 externalName 的
			if svc.Spec.Type == corev1.ServiceTypeExternalName {
				updateCh.In() <- Event{
					Type: CreateEvent,
					Obj:  obj,
				}
			}
		},
		DeleteFunc: func(obj interface{}) {
			svc := obj.(*corev1.Service)
			// 只关注 service type 为 externalName 的
			if svc.Spec.Type == corev1.ServiceTypeExternalName {
				updateCh.In() <- Event{
					Type: DeleteEvent,
					Obj:  obj,
				}
			}
		},
		UpdateFunc: func(old, cur interface{}) {
			oldSvc := old.(*corev1.Service)
			curSvc := cur.(*corev1.Service)

			// 一样则忽略
			if reflect.DeepEqual(oldSvc, curSvc) {
				return
			}

			updateCh.In() <- Event{
				Type: UpdateEvent,
				Obj:  cur,
			}
		},
	}
```

### syncQueue 

`syncQueue` 内部维护了 `workqueue` 队列, 并启动了一个 worker 协程, 该协程从内部 workqueue 获取任务, 然后调用 `syncIngress` 来同步和加载nginx配置.

代码位置: [https://github.com/kubernetes/ingress-nginx/blob/main/internal/task/queue.go](https://github.com/kubernetes/ingress-nginx/blob/main/internal/task/queue.go)

```go
// 一直调用 worker 协程, 直到 stopCh 退出, 失败重新调度间隔为 1秒.
func (t *Queue) Run(period time.Duration, stopCh <-chan struct{}) {
	wait.Until(t.worker, period, stopCh)
}

func (t *Queue) worker() {
	for {
		key, quit := t.queue.Get()
		...

		ts := time.Now().UnixNano()
		item := key.(Element)

		// 如果上次同步时间超过了 item 时间大, 则跳过, 避免无效同步.
		if item.Timestamp != 0 && t.lastSync > item.Timestamp {
			t.queue.Forget(key)
			t.queue.Done(key)
			continue
		}

		// 使用 syncIngress 来处理 key, 该方法为 ingress-nginx 核心处理入口.
		if err := t.sync(key); err != nil {
			// 同步 nginx 失败, 则重新入队
			t.queue.AddRateLimited(Element{
				Key:       item.Key,
				Timestamp: 0,
			})
		} else {
			// 成功可剔除
			t.queue.Forget(key)
			t.lastSync = ts
		}

		// 成功处理
		t.queue.Done(key)
	}
}
```

### ingress 的核心同步方法 syncIngress

syncIngress 是控制器核心处理入口, 主要调用 `OnUpdate` 和 `configureDynamically`.

1. 判断是否满足动态更新, 不满足则调用 OnUpdate 完成 nginx 配置生成, 校验和 reload 热加载.
2. 调用 configureDynamically 方法来实现动态更新.

```go
func (n *NGINXController) syncIngress(interface{}) error {
	// 从 store 的 informer 里获取 ingress 集合
	ings := n.store.ListIngresses()

	// 拿到 hosts 集合, servers 集合
	hosts, servers, pcfg := n.getConfiguration(ings)

	// 如果配置无变更, 直接跳出
	if n.runningConfig.Equal(pcfg) {
		return nil
	}

	// 不满足动态更新, 需要构建配置并 reload 加载. 
	if !utilingress.IsDynamicConfigurationEnough(pcfg, n.runningConfig) {
		klog.InfoS("Configuration changes detected, backend reload required")

		// 计算 pcfg 的 hash 值
		hash, _ := hashstructure.Hash(pcfg, &hashstructure.HashOptions{
			TagName: "json",
		})
		pcfg.ConfigurationChecksum = fmt.Sprintf("%v", hash)

		// 真正操作 nginx, 生成 nginx 配置, 语法检测, 最后执行 reload 热加载. 
		err := n.OnUpdate(*pcfg)
		if err != nil {
			return err
		}

		klog.InfoS("Backend successfully reloaded")
	}

	// 如果是首次配置, 需稍等一下
	isFirstSync := n.runningConfig.Equal(&ingress.Configuration{})
	if isFirstSync {
		klog.InfoS("Initial sync, sleeping for 1 second")
		// 首次配置, 等待个 1s, nginx 解析大配置文件时, 有可能会有延迟.
		time.Sleep(1 * time.Second)
	}

	retry := wait.Backoff{}
	retriesRemaining := retry.Steps
	err := wait.ExponentialBackoff(retry, func() (bool, error) {
		// 执行动态配置, 后面有详细讲解
		err := n.configureDynamically(pcfg)
		if err == nil {
			return true, nil
		}

		// 重试减一
		retriesRemaining--
		if retriesRemaining > 0 {
			return false, nil
		}

		// 当 err 不为 nil 时, wait 不会再进行重试调度.
		return false, err
	})

	n.runningConfig = pcfg
	return nil
}
```

### OnUpdate 真正同步 nginx 配置 

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301011115203.png)

`syncIngress` 内部会调用 `OnUpdate` 方法操作 nginx, 也就是说 `OnUpdate` 会真正的操作的 nginx 配置.

源码分析原理如下:

1. 通过模板生成 nginx 配置, 赋值给 content 对象里 ;
2. 根据不同的 collector 类型从 zipkin, jaeger 选定模板, 然后创建 `opentracing` 配置, 把 opentracing 配置写到 /etc/nginx/opentracing.json 里 ;
3. 使用 `nginx -t -c tmpfile` 来检测临时生成的 nginx 配置文件 ;
4. 当日志级别允许 level 2 时, 允许则打印新旧配置差异的部分 ;
5. 把 nginx 配置写到 /etc/nginx/nginx.conf 里 ;
6. 使用 `nginx -s reload` 进行配置的热加载.

```go
func (n *NGINXController) OnUpdate(ingressCfg ingress.Configuration) error {
	cfg := n.store.GetBackendConfiguration()
	cfg.Resolver = n.resolver

	// 通过模板生成 nginx 配置, 赋值给 content 对象里.
	content, err := n.generateTemplate(cfg, ingressCfg)
	if err != nil {
		return err
	}

	// 根据不同的 collector 类型从 zipkin, jaeger 等选定模板, 然后创建 `opentracing` 配置, 把 opentracing 配置写到 /etc/nginx/opentracing.json 里.
	err = createOpentracingCfg(cfg)
	if err != nil {
		return err
	}

	// 使用 `nginx -t -c tmpfile` 来检测临时生成的 nginx 配置文件.
	err = n.testTemplate(content)
	if err != nil {
		return err
	}

	// 判断日志级别是否允许 2 level, 允许则打印新旧配置差异的部分.
	if klog.V(2).Enabled() {
		src, _ := os.ReadFile(cfgPath)
		// 如果当前配置跟预期配置不同的话, 获取差异部分并打印输出.
		if !bytes.Equal(src, content) {
			// 把配置放到临时文件里
			tmpfile, err := os.CreateTemp("", "new-nginx-cfg")
			os.WriteFile(tmpfile.Name(), content, file.ReadWriteByUser)

			// 使用 diff 命令判断配置文件的差异
			diffOutput, err := exec.Command("diff", "-I", "'# Configuration.*'", "-u", cfgPath, tmpfile.Name()).CombinedOutput()

			// 打印配置中有差异的部分
			klog.InfoS("NGINX configuration change", "diff", string(diffOutput))

			// 删除临时文件
			os.Remove(tmpfile.Name())
		}
	}

	// 把 nginx 配置写到 /etc/nginx/nginx.conf 里.
	err = os.WriteFile(cfgPath, content, file.ReadWriteByUser)
	if err != nil {
		return err
	}

	// 使用 nginx -s reload 进行热加载 
	o, err := n.command.ExecCommand("-s", "reload").CombinedOutput()
	if err != nil {
		return fmt.Errorf("%v\n%v", err, string(o))
	}

	return nil
}
```

#### 通过模板生成 nginx 配置

在实例化控制器时 store 通过 `ngx_config.NewDefault()` 获取默认的 nginx 配置变量, 然后读取 nginx.tmpl 模板且生成模板解释器. 

`generateTemplate` 的作用是填充或者覆盖默认的 nginx 变量, 最后通过模板解释器和模板变量生成最终的配置文件.

```go
// generateTemplate returns the nginx configuration file content
func (n NGINXController) generateTemplate(cfg ngx_config.Configuration, ingressCfg ingress.Configuration) ([]byte, error) {
	// 处理 ssl 配置
	if n.cfg.EnableSSLPassthrough {
		servers := []*tcpproxy.TCPServer{}
		for _, pb := range ingressCfg.PassthroughBackends {
			svc := pb.Service
			if svc == nil {
				continue
			}
			port, err := strconv.Atoi(pb.Port.String()) // #nosec
			servers = append(servers, &tcpproxy.TCPServer{
				Hostname:      pb.Hostname,
				IP:            svc.Spec.ClusterIP,
				Port:          port,
				ProxyProtocol: false,
			})
		}

		n.Proxy.ServerList = servers
	}

	// 设置 nginx 的 server_names_hash_bucket_size, 通过预设 hash bucket 的个数, 来加快查询速度, 减少拉链查询的概率.
	nameHashBucketSize := nginxHashBucketSize(longestName)
	if cfg.ServerNameHashBucketSize < nameHashBucketSize {
		cfg.ServerNameHashBucketSize = nameHashBucketSize
	}

	// 设置 nginx 的 map_hash_bucket_size, 搭配上面的配置使用, 提高 nginx hashmap 检索速度.
	serverNameHashMaxSize := nextPowerOf2(serverNameBytes)
	if cfg.ServerNameHashMaxSize < serverNameHashMaxSize {
		cfg.ServerNameHashMaxSize = serverNameHashMaxSize
	}

	// 设置 nginx 的 worker_rlimit_nofile 每个 worker 可以打开的最大文件描述符数量, 这里的 fd 不仅仅指文件, 还是链接文件描述符 (socket fd).
	if cfg.MaxWorkerOpenFiles == 0 {
		maxOpenFiles := rlimitMaxNumFiles() - 1024
		if maxOpenFiles < 1024 {
			maxOpenFiles = 1024 // 最小为 1024
		}
		cfg.MaxWorkerOpenFiles = maxOpenFiles
	}

	// 配置 nginx worker_connections, 每个 worker 的连接数量, worker_connections 通常要小于 worker_rlimit_nofile, 一个是连接的限制, 一个是所有 fd 的限制.
	if cfg.MaxWorkerConnections == 0 {
		maxWorkerConnections := int(float64(cfg.MaxWorkerOpenFiles * 3.0 / 4))
		cfg.MaxWorkerConnections = maxWorkerConnections
	}

	// 配置转发请求时的 proxy header, 这个从 configmap 里获取.
	setHeaders := map[string]string{}
	if cfg.ProxySetHeaders != "" {
		cmap, err := n.store.GetConfigMap(cfg.ProxySetHeaders)
		if err != nil {
		} else {
			setHeaders = cmap.Data
		}
	}

	// 响应时填充的 header, 同样从 configmap 获取.
	addHeaders := map[string]string{}
	if cfg.AddHeaders != "" {
		cmap, err := n.store.GetConfigMap(cfg.AddHeaders)
		if err != nil {
		} else {
			addHeaders = cmap.Data
		}
	}

	// 设置 ssl 参数
	cfg.SSLDHParam = sslDHParam
	cfg.DefaultSSLCertificate = n.getDefaultSSLCertificate()

	// 配置 access_log 和 error_log 的位置
	if n.cfg.IsChroot {
		if cfg.AccessLogPath == "/var/log/nginx/access.log" {
			cfg.AccessLogPath = fmt.Sprintf("syslog:server=%s", n.cfg.InternalLoggerAddress)
		}
		if cfg.ErrorLogPath == "/var/log/nginx/error.log" {
			cfg.ErrorLogPath = fmt.Sprintf("syslog:server=%s", n.cfg.InternalLoggerAddress)
		}
	}

	tc := ngx_config.TemplateConfig{
		// 配置请求时自定义的 header 填充, 模板中使用 proxy_set_header 指令 
		ProxySetHeaders:          setHeaders,

		// 定制响应报文的 header, 模板中使用 more_set_headers 指令
		AddHeaders:               addHeaders,

		// 连接全队列的大小, 从 /net/core/somaxconn 获取, 读取失败或者过小则设置为 511.
		BacklogSize:              sysctlSomaxconn(),
		Backends:                 ingressCfg.Backends,

		// nginx 的 server 段配置
		Servers:                  ingressCfg.Servers,

		// nginx stream tcp server 的配置
		TCPBackends:              ingressCfg.TCPEndpoints,
		// nginx stream udp server 的配置
		UDPBackends:              ingressCfg.UDPEndpoints,

		// ngx_config 配置
		Cfg:                      cfg,

		// redirect 跳转
		RedirectServers:          utilingress.BuildRedirects(ingressCfg.Servers),

		// 监听的端口
		ListenPorts:              n.cfg.ListenPorts,

		// 开启 metrics 监控, 这个是在 http lua 逻辑里
		EnableMetrics:            n.cfg.EnableMetrics,

		// 主要跟 geolite 有关系
		MaxmindEditionFiles:      n.cfg.MaxmindEditionFiles,

		// 在每个 http server 里加入一个 location path 为 /healthz 的接口, 处理逻辑是直接 return  200
		HealthzURI:               nginx.HealthPath,

		// 声明下 nginx pid 位置在 `/tmp/nginx/nginx.pid`
		PID:                      nginx.PID,

		// 开启 nginx status 接口
		StatusPath:               nginx.StatusPath,
		StatusPort:               nginx.StatusPort,
	}

	// 在 nginx.conf 文件头部位置的注释里加入配置的 checksum 校验码, 其实就是配置文件的 hash 值.
	tc.Cfg.Checksum = ingressCfg.ConfigurationChecksum

	// n.t 为 nginx.tmpl 的模板解释器, write 通过传递的 ngx 变量来生成 nginx 配置.
	return n.t.Write(tc)
}
```

`nginx.tmpl` 模板语言为 golang 标准库 `text/template` 的语法. http 和 http2 的 server 段在 nginx.tmpl 实在太冗长了.

这里就简单举例说明 nginx stream tcp 和 udp 的模板构建过程.

```go
stream {
    # 定义 lua 包位置
    lua_package_path "/etc/nginx/lua/?.lua;/etc/nginx/lua/vendor/?.lua;;";

    # 设置一个共享存储
    lua_shared_dict tcp_udp_configuration_data 5M;

    # 判断 access_log 是否开启 access_log, 如果开启则指定文件路径及格式.
    {{ if or $cfg.DisableAccessLog $cfg.DisableStreamAccessLog }}
    access_log off;
    {{ else }}
    access_log {{ or $cfg.StreamAccessLogPath $cfg.AccessLogPath }} log_stream {{ $cfg.AccessLogParams }};
    {{ end }}

    # 错误日志
    error_log  {{ $cfg.ErrorLogPath }} {{ $cfg.ErrorLogLevel }};

    upstream upstream_balancer {
	    # 这个只是占位符而已
        server 0.0.0.1:1234; # placeholder

        # tcp 和 udp 转发是依赖 openretry balancer_by_lua_block 控制的.
        # 具体转发逻辑在 tcp_udp_balancer.lua 这里实现的, balance 是调度入口.
        balancer_by_lua_block {
          tcp_udp_balancer.balance()
        }
    }

    # 遍历生成 TCP services
    {{ range $tcpServer := .TCPBackends }}
    server {
	    # 针对 tcp ipv4 listen 进行渲染
        {{ range $address := $all.Cfg.BindAddressIpv4 }}
        listen                  {{ $address }}:{{ $tcpServer.Port }}{{ if $tcpServer.Backend.ProxyProtocol.Decode }} proxy_protocol{{ end }};
        {{ else }}
        listen                  {{ $tcpServer.Port }}{{ if $tcpServer.Backend.ProxyProtocol.Decode }} proxy_protocol{{ end }};
        {{ end }}

	    # 针对 tcp ipv6 listen 进行渲染
        {{ if $IsIPV6Enabled }}
        {{ range $address := $all.Cfg.BindAddressIpv6 }}
        listen                  {{ $address }}:{{ $tcpServer.Port }}{{ if $tcpServer.Backend.ProxyProtocol.Decode }} proxy_protocol{{ end }};
        {{ else }}
        listen                  [::]:{{ $tcpServer.Port }}{{ if $tcpServer.Backend.ProxyProtocol.Decode }} proxy_protocol{{ end }};
        {{ end }}
        {{ end }}

	    # 配置 tcp proxy 参数
        proxy_timeout           {{ $cfg.ProxyStreamTimeout }};
        proxy_next_upstream     {{ if $cfg.ProxyStreamNextUpstream }}on{{ else }}off{{ end }};
        proxy_next_upstream_timeout {{ $cfg.ProxyStreamNextUpstreamTimeout }};
        proxy_next_upstream_tries   {{ $cfg.ProxyStreamNextUpstreamTries }};

	    # 转发到 upstream_balancer
        proxy_pass              upstream_balancer;

	    # proxy_protocol 是在转发的 tcp 报文中插入客户端 ip 地址, 这样经过层层网关转发后, 后面的 nginx 也可以拿到客户端地址.
        {{ if $tcpServer.Backend.ProxyProtocol.Encode }}
        proxy_protocol          on;
        {{ end }}
    }
    {{ end }}

    # 遍历生成 UDP services
    {{ range $udpServer := .UDPBackends }}
    server {
        {{ range $address := $all.Cfg.BindAddressIpv4 }}
        listen                  {{ $address }}:{{ $udpServer.Port }} udp;
        {{ else }}
        listen                  {{ $udpServer.Port }} udp;
        {{ end }}
        {{ if $IsIPV6Enabled }}
        {{ range $address := $all.Cfg.BindAddressIpv6 }}
        listen                  {{ $address }}:{{ $udpServer.Port }} udp;
        {{ else }}
        listen                  [::]:{{ $udpServer.Port }} udp;
        {{ end }}
        {{ end }}

        proxy_responses         {{ $cfg.ProxyStreamResponses }};
        proxy_timeout           {{ $cfg.ProxyStreamTimeout }};
        proxy_next_upstream     {{ if $cfg.ProxyStreamNextUpstream }}on{{ else }}off{{ end }};
        proxy_next_upstream_timeout {{ $cfg.ProxyStreamNextUpstreamTimeout }};
        proxy_next_upstream_tries   {{ $cfg.ProxyStreamNextUpstreamTries }};
        proxy_pass              upstream_balancer;
    }
    {{ end }}
}
```

stream 的 backend 调度选择是依赖 tcp_udp_balancer.lua 来实现的. 

在 nginx stream 插入了 `balancer_by_lua_block` 调度块, 当 nginx 对该 server 进行转发时, 先通过 balance 调度算法获取理想的 peer 地址, 然后 nginx 依赖自己的 stream 模块跟 peer 建连然后数据转发.

通过源码分析得知, balance 内部维护的地址池不是 service 的 clusterIP 集群地址, 而是每个 endpoint 的地址. 这样减少了 ipvs 和 iptables 的 dnat/snat 开销, 并且 balancer 内部提供了更细致的负载均衡算法.

```lua
function _M.balance()
  # 获取负载均衡器对象
  local balancer = get_balancer()
  if not balancer then
    return
  end

  # 经过调度算法来获取转发的地址
  local peer = balancer:balance()
  if not peer then
    return
  end

  # 把获取的 peer 地址放到 ngx 变量里, nginx 依赖该变量进行转发.
  local ok, err = ngx_balancer.set_current_peer(peer)
  if not ok then
  end
end
```

nginx.tmpl 文件长度差不多有 `1500` 行, 大家可以参照传入的变量结构体 `ngx_config.TemplateConfig` 来分析下模板生成的过程.

`nginx.tmpl` 模板的源码位置:

[https://github.com/kubernetes/ingress-nginx/blob/3aa53aaf5b210dd937598928e172ef1478e90e69/rootfs/etc/nginx/template/nginx.tmpl](https://github.com/kubernetes/ingress-nginx/blob/3aa53aaf5b210dd937598928e172ef1478e90e69/rootfs/etc/nginx/template/nginx.tmpl).

#### 创建 opentracing 配置

判断 collector 类型, 然后实例化不同的收集器的模板, 生成最后的配置后写到 `/etc/nginx/opentracing.json` 目录里.

```go
const zipkinTmpl = `{
	...
}`

const jaegerTmpl = `{
  "service_name": "{{ .JaegerServiceName }}",
  "propagation_format": "{{ .JaegerPropagationFormat }}",
  "sampler": {
	"type": "{{ .JaegerSamplerType }}",
	"param": {{ .JaegerSamplerParam }},
	"samplingServerURL": "{{ .JaegerSamplerHost }}:{{ .JaegerSamplerPort }}/sampling"
  },
  "reporter": {
	"endpoint": "{{ .JaegerEndpoint }}",
	"localAgentHostPort": "{{ .JaegerCollectorHost }}:{{ .JaegerCollectorPort }}"
  },
  "headers": {
	"TraceContextHeaderName": "{{ .JaegerTraceContextHeaderName }}",
	"jaegerDebugHeader": "{{ .JaegerDebugHeader }}",
	"jaegerBaggageHeader": "{{ .JaegerBaggageHeader }}",
	"traceBaggageHeaderPrefix": "{{ .JaegerTraceBaggageHeaderPrefix }}"
  }
}`

func createOpentracingCfg(cfg ngx_config.Configuration) error {
	var tmpl *template.Template
	var err error

	if cfg.ZipkinCollectorHost != "" {
		tmpl, err = template.New("zipkin").Parse(zipkinTmpl)
	} else if cfg.JaegerCollectorHost != "" || cfg.JaegerEndpoint != "" {
		tmpl, err = template.New("jaeger").Parse(jaegerTmpl)
	} else if cfg.DatadogCollectorHost != "" {
		tmpl, err = template.New("datadog").Parse(datadogTmpl)
	}
	...

	tmplBuf := bytes.NewBuffer(make([]byte, 0))
	err = tmpl.Execute(tmplBuf, cfg)
	if err != nil {
		return err
	}

	expanded := os.ExpandEnv(tmplBuf.String())
	return os.WriteFile("/etc/nginx/opentracing.json", []byte(expanded), file.ReadWriteByUser)
}
```

nginx 通过模板生成配置时, 会在配置中开启 `opentracing`, 并声明各个收集器的动态库及相关配置的位置.

```c
load_module /etc/nginx/modules/ngx_http_opentracing_module.so;
opentracing on;
opentracing_propagate_context;
# opentracing_load_tracer /usr/local/lib/libdd_opentracing.so /etc/nginx/opentracing.json;
# opentracing_load_tracer /usr/local/lib/libjaegertracing_plugin.so /etc/nginx/opentracing.json;
opentracing_load_tracer /usr/local/lib/libzipkin_opentracing_plugin.so /etc/nginx/opentracing.json;
...
```

官方的 nginx 自身是不支持该 opentracing 全链路追踪, 依赖 `nginx-opentracing` 项目以插件 plugin 形式来让 nginx 支持 opentracing 链路追踪功能.

项目地址: [https://github.com/opentracing-contrib/nginx-opentracing](https://github.com/opentracing-contrib/nginx-opentracing)

#### 测试并校验 nginx 配置

创建一个临时的文件, 把 nginx 配置写到临时文件, 然后使用 `nginx -t -c nginx-cfg-xxx` 测试下. 

但让人奇怪的是只是通过 exec 的 err 判断是否错误, 而不是通过输出的内存或者 process exit code 来识别错误. 想来是因为维护者因为前期做了很多校验, 后面又是通过模板来生成的配置, 假定不出问题吧.

```go
func (n NGINXController) testTemplate(cfg []byte) error {
	tmpDir := os.TempDir() + "/nginx"

	// 在 /tmp/nginx 临时目录创建前缀为 `nginx-cfg` 后跟随机数的临时文件.
	tmpfile, err := os.CreateTemp(tmpDir, tempNginxPattern)
	if err != nil {
		return err
	}
	defer tmpfile.Close()

	err = os.WriteFile(tmpfile.Name(), cfg, file.ReadWriteByUser)
	if err != nil {
		return err
	}

	// 通过 nginx -t -c nginx-cfgxxxx 来测试临时配置是否合法.
	out, err := n.command.Test(tmpfile.Name())
	if err != nil {
		return errors.New(oe)
	}

	os.Remove(tmpfile.Name())
	return nil
}
```

nginxCommand 库封装了 nginx 的一些命令, 比如下面的 `nginx -t`.

```go
func (nc NginxCommand) Test(cfg string) ([]byte, error) {
	return exec.Command(nc.Binary, "-c", cfg, "-t").CombinedOutput()
}
```

#### nginx reload 设计实现

由于篇幅原因不分析 nginx 代码了, 下面是 `nginx -s reload` 的实现原理过程.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/202212311601328.png)

1. 修改完 nginx 的配置文件后, 向 master 进程发送 HUP 信号, `nginx -s reload` 本质也是发送 HUP 信号;
2. master 进程在收到 HUP 信号以后, 会在第二步检查我们的配置文件语法是否正确;
3. 在 nginx 的配置语法全部正确以后, master 进程会打开新的监听端口 ;
4. master 进程会用新的 nginx.conf 配置文件来启动新的 worker 子进程 ;
5. 由 master 进程再向老 worker 子进程发送 QUIT 信号.

**老 worker 如何优雅退出 ?**

nginx 是可以实现优雅退出的, 老 worker 收到 SIGQUIT 信号后不会立马退出, 而是先在本 worker eventloop 上摘除 listenfd 的监听, 这样 epoll 就拿不到 listenfd 的事件, 更不会去做 accept 操作了. 之后 worker 等待该 worker 上的所有长连接都关闭才退出.

这里说的长连接说的是 tcp stream 和 http2 这类长连接, 另外当一个 http 请求迟迟不返回 response, 也是影响 worker 退出.

**如何老连接就是不退出怎么办 ?**

在 nginx v1.11.11 版本之前是干等, 客户端长连接不断, 那么 worker 进程就不退出.

之后版本加入了 `worker_shutdown_timeout` 参数, 等待一段时间还连接还未退出, 则强制主动关闭.

> 实际上 worker_shutdown_timeout 参数的实现原理还是有些绕的, 后面补写一篇文章专门分析下 nginx grace reload 细节.

### 如何动态修改 nginx 配置, 非 reload (configureDynamically)

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301011133932.png)

通过前面 nginx.tmpl 的配置得知, nginx (openretry) 转发的逻辑是依赖 upstream 的 balancer_by_lua_block 指令实现的.

http 和 stream (tcp/udp) 在生成配置时, 在 upstream 段里都插入了 `balancer_by_lua_block` 指令用来实现自定义负载均衡逻辑, nginx 会依赖该 balancer 来获取转发的地址, 然后对该连接进行转发.

> 该 lua 转发模块代码位置是 `rootfs/etc/nginx/lua/balancer.lua`.

#### balancer_by_lua_block 是怎么回事 ?

`balancer_by_lua_block` 是一个支持自定义负载均衡器的指令, 通常基于 nginx 的服务发现就是通过该指令实现的.

开发时一定要注意事项, balancer_by_lua_block 只是通过自定义负载均衡算法获取 peer 后端地址, 接着通过 `balancer.set_current_peer(ip, port)` 进行赋值. 后面连接的建立，连接池维护，数据拷贝转发等流程统统不在这里，而是由 nginx 内部 upstream 转发逻辑实现.

一句话，nginx 只是调用 `balancer_by_lua_block` 获取理想的后端地址而已.

下面是使用 `balancer_by_lua_block` 实现调度地址池的例子:

```c
upstream backend{
    server 0.0.0.0;
    
    balancer_by_lua_block {
        local balancer = require "ngx.balancer"
        local host = {"s1.xiaorui.cc", "s2.xiaorui.cc"}
        local backend = ""
        local port = ngx.var.server_port
        local remote_ip = ngx.var.remote_addr
        local key = remote_ip..port
        
        # 使用地址 hash 调度算法
        local hash = ngx.crc32_long(key);
        hash = (hash % 2) + 1
        backend = host[hash]
        ngx.log(ngx.DEBUG, "ip_hash=", ngx.var.remote_addr, " hash=", hash, " up=", backend, ":", port)
        
        # 配置后端地址, nginx 进行转发时依赖该地址
        local ok, err = balancer.set_current_peer(backend, port)
        if not ok then
            ngx.log(ngx.ERR, "failed to set the current peer: ", err)
            return ngx.exit(500)
        end
    }
}

server {
	listen 80;
	server_name xiaorui.cc
	location / {
		proxy_pass http://backend;
	}
}
```

lua-nginx-module 项目中关于 balancer_by_lua_block 实现:

[https://github.com/openresty/lua-nginx-module#balancer_by_lua_block](https://github.com/openresty/lua-nginx-module#balancer_by_lua_block)

#### 在 nginx 里加入 balancer_by_lua_block 指令

在 `nginx.tmpl` 中加入了 `balancer_by_lua_block` 指令, 所以不管是 http 和 stream 段里的 upstream 转发, 不再走 server 配置, 而是走 `balancer_by_lua_block` 自定义流程.

```c
http {
    upstream upstream_balancer {
	    // 只是占位符, openretry 优先走 balancer_by_lua 逻辑块.
        server 0.0.0.1; # placeholder

        balancer_by_lua_block {
          balancer.balance()
        }

        {{ if (gt $cfg.UpstreamKeepaliveConnections 0) }}
        keepalive {{ $cfg.UpstreamKeepaliveConnections }};
        keepalive_time {{ $cfg.UpstreamKeepaliveTime }};
	...
        {{ end }}
    }

    ...

    server {
	...
    }
}

stream {
    upstream upstream_balancer {
	    // 同上, 只是占位符, 避免 nginx -t 检测出错.
        server 0.0.0.1:1234; # placeholder

        balancer_by_lua_block {
          tcp_udp_balancer.balance()
        }
    }

    ...

    server {
	...
    }
}

```

#### 把变更信息通知给 nginx

- 检查 http backends 是否有变更, 当有变更时, 把 backends 数据通知给 nginx 的 `http://127.0.0.1:10246/configuration/backends` 接口上.
- 检查 tcp/udp strem backends 是否有变更, 发生变更时, 把 stream backends 数据发到 nginx 的 tcp `10247` 端口上.
- 当证书发生变更时, 发数据发到 nginx 的 `http://127.0.0.1:10246/configuration/servers` 接口上.

```go
func (n *NGINXController) configureDynamically(pcfg *ingress.Configuration) error {
	backendsChanged := !reflect.DeepEqual(n.runningConfig.Backends, pcfg.Backends)
	// 当 endpoints 地址发生变更时
	if backendsChanged {
		// 动态修改 http 的 backends
		err := configureBackends(pcfg.Backends)
		if err != nil {
			return err
		}
	}

	streamConfigurationChanged := !reflect.DeepEqual(n.runningConfig.TCPEndpoints, pcfg.TCPEndpoints) || !reflect.DeepEqual(n.runningConfig.UDPEndpoints, pcfg.UDPEndpoints)
	// 当 endpoints 地址发生变更时
	if streamConfigurationChanged {
		// 动态修改 tcp 和 udp 的 backends 地址列表
		err := updateStreamConfiguration(pcfg.TCPEndpoints, pcfg.UDPEndpoints)
		if err != nil {
			return err
		}
	}

	serversChanged := !reflect.DeepEqual(n.runningConfig.Servers, pcfg.Servers)
	// 当 servers 地址发生变更时
	if serversChanged {
		// 动态修改证书相关配置
		err := configureCertificates(pcfg.Servers)
		if err != nil {
			return err
		}
	}

	return nil
}
```

这里拿 `configureBackends()` 变更配置来说. 组装 openresty 专用的 backends 数据, 然后序列化成 json, post 发给 openresty 的 `/configuration/backends` 接口上.

```go
func configureBackends(rawBackends []*ingress.Backend) error {
	backends := make([]*ingress.Backend, len(rawBackends))

	for i, backend := range rawBackends {
		luaBackend := &ingress.Backend{
			...
		}

		var endpoints []ingress.Endpoint
		for _, endpoint := range backend.Endpoints {
			endpoints = append(endpoints, ingress.Endpoint{
				Address: endpoint.Address,
				Port:    endpoint.Port,
			})
		}

		luaBackend.Endpoints = endpoints
		backends[i] = luaBackend
	}

	statusCode, _, err := nginx.NewPostStatusRequest("/configuration/backends", "application/json", backends)
	if err != nil {
		return err
	}

	if statusCode != http.StatusCreated {
		return fmt.Errorf("unexpected error code: %d", statusCode)
	}

	return nil
}

func NewPostStatusRequest(path, contentType string, data interface{}) (int, []byte, error) {
	url := fmt.Sprintf("http://127.0.0.1:%v%v", StatusPort, path)
	buf, err := json.Marshal(data)
	...

	res, err := client.Post(url, contentType, bytes.NewReader(buf))
	...

	body, err := io.ReadAll(res.Body)
	...
	return res.StatusCode, body, nil
}
```

上面代码是如何发送变更信息, 那么谁来接收动态数据的投递? 

`nginx.conf` 中定义了一个解决动态配置更新的 server 配置段, 其中变量 StatusPort 为 10246, 接口的 prefix 路径为 `/configuration`, 该接口定义了 content_by_lua_block 处理块.

当接口收到请求后, 调用自定义 lua 模块 `configuration.lua` 中 `configuration.call()` 入口方法.

```c
    server {
        listen 127.0.0.1:{{ .StatusPort }};

        keepalive_timeout 0;
        gzip off;

        access_log off;

        location {{ $healthzURI }} {
            return 200;
        }

        location /configuration {
            client_max_body_size                    {{ luaConfigurationRequestBodySize $cfg }};
            client_body_buffer_size                 {{ luaConfigurationRequestBodySize $cfg }};
            proxy_buffering                         off;

            content_by_lua_block {
              configuration.call()
            }
        }
    }
```

下面分析 `configuration.call()` 的实现原理. `call()` 中硬编码写了各个接口的处理方法.

当 `ngx.var.request_uri` 为 `/configuration/backends` 时候, 调用 `handle_backends` 方法处理该路由.

`handle_backends` 内部实现过程很简单, 先解析 request body, 然后把读到的 body 字符串放到共享存储 `configuration_data` 的 backends 键里, 然后更新下操作的时间戳.

`configuration_data` 是一个 ngx.shared.Dict 共享内存的字典存储结构, 其 set/get 操作是并发安全的. nx.shared.dict 内部通过红黑树实现的 hashmap, 使用 lru 实现的数据淘汰.

configuration_data:set 的时候没有 cjson 解析对象, 而是直接赋值json string.

```lua
function _M.call()
  if ngx.var.request_method ~= "POST" and ngx.var.request_method ~= "GET" then
    ngx.status = ngx.HTTP_BAD_REQUEST
    return
  end

  # 处理证书的 servers 
  if ngx.var.request_uri == "/configuration/servers" then
    handle_servers()
    return
  end

  # 处理通用配置 general
  if ngx.var.request_uri == "/configuration/general" then
    handle_general()
    return
  end

  # 处理证书的 http handler
  if ngx.var.uri == "/configuration/certs" then
    handle_certs()
    return
  end

  # 处理 backends http handler
  if ngx.var.request_uri == "/configuration/backends" then
    handle_backends()
    return
  end

  ngx.status = ngx.HTTP_NOT_FOUND
  ngx.print("Not found!")
end

local function handle_backends()
  # 获取当前 nginx 内的 backends 配置
  if ngx.var.request_method == "GET" then
    ngx.status = ngx.HTTP_OK
    ngx.print(_M.get_backends_data())
    return
  end

  # 读取 request body
  local backends = fetch_request_body()
  if not backends then
    ngx.status = ngx.HTTP_BAD_REQUEST
    return
  end

  # 把 backends 放到 ngx.shared 的 configuration_data 存储的 backends 键值里.
  local success, err = configuration_data:set("backends", backends)
  if not success then
    ngx.status = ngx.HTTP_BAD_REQUEST
    return
  end

  # 记录更新时间
  ngx.update_time()
  local raw_backends_last_synced_at = ngx.time()
  success, err = configuration_data:set("raw_backends_last_synced_at", raw_backends_last_synced_at)
  if not success then
    ngx.status = ngx.HTTP_BAD_REQUEST
    return
  end

  ngx.status = ngx.HTTP_CREATED
end
```

上面的 `handle_backends` 只是获取请求的 json body 字符串, 然后把字符串写到 ngx.shared.dict 存储里. 那么谁来读取 ? 谁来 json decode ? 

控制器在 nginx.conf 配置文件中加入了 `init_worker_by_lua_block` 初始化块, 所以当 nginx 启动时会调用 `balancer.init_worker` 进行模块初始化.

先异步执行 sync_backends_with_external_name, 同步 service 类型为 external_name 的配置, 然后每隔一秒调用一次 sync_backends 和 sync_backends_with_external_name.

代码如下: `rootfs/etc/nginx/lua/balancer.lua::init_worker()`

```lua
function _M.init_worker()
  # 通过定时器实现异步执行 sync_backends_with_external_name
  local ok, err = ngx.timer.at(0, sync_backends_with_external_name)
  if not ok then
    ngx.log(ngx.ERR, "failed to create timer: ", err)
  end

  # 每秒调用一次 sync_backends
  ok, err = ngx.timer.every(BACKENDS_SYNC_INTERVAL, sync_backends)
  if not ok then
    ngx.log(ngx.ERR, "error when setting up timer.every for sync_backends: ", err)
  end

  # 每秒调用一次 sync_backends_with_external_name
  ok, err = ngx.timer.every(BACKENDS_SYNC_INTERVAL, sync_backends_with_external_name)
  if not ok then
    ngx.log(ngx.ERR, "error when setting up timer.every for sync_backends_with_external_name: ",
            err)
  end
end
```

`sync_backends()` 被定时器周期性调度, 从 `ngx.shared.dict` 获取 backends 数据, 反序列化后, 遍历所有的 backend 对象, 依次调用 `sync_backend` 来向 balancer 同步配置.

```lua
local function sync_backends()
  # 从 ngx.shared.dict 获取 backends 键值数据
  local backends_data = configuration.get_backends_data()
  ...

  # 把 json string 进行反序列为 json object 对象
  local new_backends, err = cjson.decode(backends_data)
  ...

  # 通过 sync_backend() 处理 backend 对象
  local balancers_to_keep = {}
  for _, new_backend in ipairs(new_backends) do
    if is_backend_with_external_name(new_backend) then
      ...
    else
      # 向 balancer 同步配置
      sync_backend(new_backend)
    end
    balancers_to_keep[new_backend.name] = true
  end
end

local function sync_backend(backend)
  # 如果 endpoints 为空, 则跳出.
  if not backend.endpoints or #backend.endpoints == 0 then
    balancers[backend.name] = nil
    return
  end

  # 简化 endpoints 数据结构, 把复杂的 struct 转成 lua table 数组.
  backend.endpoints = format_ipv6_endpoints(backend.endpoints)

  local implementation = get_implementation(backend)
  local balancer = balancers[backend.name]

  # 该 name 没有 balancer 对象, 就创建一个
  if not balancer then
    balancers[backend.name] = implementation:new(backend)
    return
  end

  # 调用 balancer 的 sync 方法, 把 backend.endpoints 更新到 peers 对象里.
  balancer:sync(backend)
end
```

`ingress-nginx` 当前实现了下面几种负载均衡算法. 默认算法为 `round_robin` 轮询. 这些 lua 算法模块位置在 `rootfs/etc/nginx`. 可以发现负载均衡算法中没有 `least_conn` 最少连接算法, 也没有 `p2c` (自适应负载均衡算法) 算法.

为什么不能使用 nginx 自带的负载均衡算法, 而是在 lua 中实现 ? 这是因为 ingress-nginx 是通过 `balancer_by_lua` 实现的地址池的负载均衡. 另外 `balancer_by_lua` 内部没有实现健康检查, 而是通过 informer 监听 apiserver 感知地址的变更.

```c
sticky_persistent.lua
sticky_balanced.lua
sticky.lua
round_robin.lua
resty.lua
ewma.lua
chashsubset.lua
chash.lua
```

#### 为什么需要动态更新 upstream backends ? 

不管是 nginx 和 openresty 都只支持配置的 reload 热加载, 不支持动态更新的. 但社区中基于 openresty 的 kong 和 apisix 都支持多源的动态更新配置, 社区中也有支持动态更新的 lua 模块可以使用.

当 nginx 作为 ingress 角色时, 遇到频繁变更 service endpoints 的场景下, nginx reload 开销不会小的, 每次都需要 new worker 及 kill old worker, 旧 worker 的长请求不断又是个问题. 新 worker 是新的子进程没法继承旧 worker 的连接池, 所以需要重新建连连接和维护 upstream 连接池, 这都会影响性能和时延 latency.

如果 nginx 不支持动态更新, 在一个大集群的的上下线会引发 ingress-nginx 不断的 reload.

在 ingress 支持 upstream 和证书动态更新后, 新配置加载的开销会小很多. 只需要把更新的配置通知给 openresty 的动态配置接口就可以了. balancer.lua 模块会维护每个 backend 地址池的负载均衡逻辑.

## 总结 

ingress-nginx 的实现原理, 就是控制器监听 kube-apiserver 资源更新, 当有资源更新时, 通过 nginx 配置模板生成配置文件, 然后 reload 热加载的过程. 

为了应对 kubernetes endpoints 变更引发的 nginx 频繁 reload,  所以 ingress-nginx 在 nginx 里使用 lua 实现了配置热更新的功能, 主要是针对地址池构建各负载算法的 balancer, 当 nginx location upstream 进行转发钱, 先从 balancer_by_lua 里获取转发的后端地址, 然后 nginx 再对该后端地址进行转发.   