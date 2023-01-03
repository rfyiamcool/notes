## 源码分析 kubelet pod 生成 coredns resolv.conf 配置原理

本文分析 k8s pod 容器内的域名解析配置文件 `resolv.conf` 如何生成, 被如何在容器内创建.

- kubernetes pod dnsPolicy 策略
- kubelet 内的 clusterDNS 配置是怎么拿到的
- 源码分析 k8s pod resolv.conf 生成原理

### kubernetes pod dnsPolicy 策略

在分析 kubelet 如何为 pod 生成 resolv.conf 之前, 需要先理解 pod dnspolicy 几种策略.

Kubernetes 中 Pod 的 DNS 策略有四种类型:

- Default: Pod 使用宿主机的 DNS 配置 ;
- ClusterFirst: K8s 的默认设置, 先在 K8s 集群配置的 coreDNS 中查询，查不到的再去继承自主机的上游 nameserver 中查询 ;
- ClusterFirstWithHostNet: 对于网络配置为 hostNetwork 的 Pod 而言，其 DNS 配置规则与 ClusterFirst 一致 ;
- None: 忽略 K8s 环境的 DNS 配置, 只认 Pod 里的 dnsConfig 配置项.

#### ClusterFirstWithHostNet

使用 k8s 的 coredns server 配置, 但使用主机协议栈.

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx

   hostNetwork: true
   dnsPolicy: ClusterFirstWithHostNet
```

#### ClusterFirst

使用 k8s 的 coredns servers 配置.

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: ClusterFirst
```

#### Default

使用宿主机 dns 配置.

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
   dnsPolicy: Default
```

#### None

不适用 k8s 和主机的 dns 配置, 而是使用 dnsConfig 配置段.

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

#### kubelet 内的 clusterDNS 配置是怎么拿到的 ?

Kubelet 启动时可绑定 `--cluster-dns` 参数, 该参数会作为 kubelet 的 ClusterDNS 配置, 当 pod 的 dns 策略为 `ClusterFirst` 时, 则使用 kubelet 的 ClusterDNS 配置. 通常该参数指定为 coredns 实例的地址.

```go
func AddKubeletConfigFlags() {
	...

	fs.StringSliceVar(&c.ClusterDNS, "cluster-dns", c.ClusterDNS, "Comma-separated list of DNS server IP address.  This value is used for containers DNS server in case of Pods with \"dnsPolicy=ClusterFirst\". Note: all DNS servers appearing in the list MUST serve the same set of records otherwise name resolution within the cluster may not work correctly. There is no guarantee as to which DNS server may be contacted for name resolution.")

	...
}

func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	...
	clusterDNS := make([]net.IP, 0, len(kubeCfg.ClusterDNS))
	for _, ipEntry := range kubeCfg.ClusterDNS {
		clusterDNS = append(clusterDNS, ip)
	}
	...
}
```

### 源码分析 k8s pod resolv.conf 生成原理 ? 

首先 kubelet 从 apiserver 收到事件后, 调用 `SyncPod` 来创建 pod, pod 里容器的 dns 的配置是在 `pause` 里的.  `createPodSandbox` 方法用来创建 pod, 其内部会生成 pause 所需的配置, 然后调用 docker cri 来传递配置并创建 pause 容器. 

代码位置: `pkg/kubelet/kuberuntime/kuberuntime_manager.go`

```go
func (m *kubeGenericRuntimeManager) createPodSandbox(ctx context.Context, pod *v1.Pod, attempt uint32) (string, string, error) {
	// 生成 pause 的配置
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, attempt)
	if err != nil {
		return "", message, err
	}

	// 创建 pods 的日志目录
	err = m.osInterface.MkdirAll(podSandboxConfig.LogDirectory, 0755)
	if err != nil {
		return "", message, err
	}

	...

	// 启动 pause 容器
	podSandBoxID, err := m.runtimeService.RunPodSandbox(ctx, podSandboxConfig, runtimeHandler)
	if err != nil {
		return "", message, err
	}

	return podSandBoxID, "", nil
}
```

#### 生成配置

`generatePodSandboxConfig` 方法用来生成 pod 配置, 其中有我们关注的 dns 配置项.

代码位置: `pkg/kubelet/kuberuntime/kuberuntime_sandbox.go`

```go
func (m *kubeGenericRuntimeManager) generatePodSandboxConfig(pod *v1.Pod, attempt uint32) (*runtimeapi.PodSandboxConfig, error) {
	podUID := string(pod.UID)
	podSandboxConfig := &runtimeapi.PodSandboxConfig{
		...
	}

	// 获取 dns
	dnsConfig, err := m.runtimeHelper.GetPodDNS(pod)
	if err != nil {
		return nil, err
	}
	podSandboxConfig.DnsConfig = dnsConfig

	...

	return podSandboxConfig, nil
}
```

`GetPodDNS` 用来获取创建容器所需的 dnsConfig 配置, 其内部根据 pod 的 dnsPolicy 获取不同的 dns 配置.

每种 dnsPolicy 使用不同的配置策略:

- 如果为 `ClusterFirst` 和 `ClusterFirstWithHostNet` 都使用 coredns server 地址 ;
- 如果为 `Default`, 则使用 pod 所在的宿主机的配置 ;
- 如果为 `None`, 则使用 pod spec 中的 dnsConfig 配置.

代码流程如下:

代码位置: `pkg/kubelet/network/dns/dns.go`

```go
// GetPodDNS returns DNS settings for the pod.
func (c *Configurer) GetPodDNS(pod *v1.Pod) (*runtimeapi.DNSConfig, error) {
	// 获取当前 host 的 dns config, 其实就是解析 resolv.conf
	dnsConfig, err := c.getHostDNSConfig()
	if err != nil {
		return nil, err
	}

	// 获取 pod 的 dns 策略
	dnsType, err := getPodDNSType(pod)
	if err != nil {
		// 默认使用 dns cluster 模式.
		dnsType = podDNSCluster
	}
	switch dnsType {
	case podDNSNone:
		// 这里不配置, 最后面会使用 `appendDNSConfig` 把 pod.Sepc.DNSConfig 合并到 dnsConfig 里.
		dnsConfig = &runtimeapi.DNSConfig{}
	case podDNSCluster:
		// 如果 pod dnsPolicy 为 cluster 模式
		if len(c.clusterDNS) != 0 {
			// 把 kubelet 启动时参数 --cluster-dns 地址加入到 dnsConfig.Servers
			dnsConfig.Servers = []string{}
			for _, ip := range c.clusterDNS {
				dnsConfig.Servers = append(dnsConfig.Servers, ip.String())
			}

			// 把 dns search domain 加入到 config.Searches
			dnsConfig.Searches = c.generateSearchesForDNSClusterFirst(dnsConfig.Searches, pod)

			// 配置 dns options, 默认 ndots:5
			dnsConfig.Options = defaultDNSOptions
			break
		}
		fallthrough
	case podDNSHost:
		// 如果 pod dnsPolicy 是 Default 默认策略, 且当前 kubelet 实例读取 resolv.conf 为空, 那么则使用 127.0.0.1 本地地址作为 server.
		if c.ResolverConfig == "" {
			for _, nodeIP := range c.nodeIPs {
				if utilnet.IsIPv6(nodeIP) {
					dnsConfig.Servers = append(dnsConfig.Servers, "::1")
				} else {
					dnsConfig.Servers = append(dnsConfig.Servers, "127.0.0.1")
				}
			}
			if len(dnsConfig.Servers) == 0 {
				dnsConfig.Servers = append(dnsConfig.Servers, "127.0.0.1")
			}
			dnsConfig.Searches = []string{"."}
		}
	}

	// 如果 pod 有配置专门的 dnsConfig 选项, 则跟 dnsConfig 拼凑在一起, 主要配置 servers, search, options.
	if pod.Spec.DNSConfig != nil {
		dnsConfig = appendDNSConfig(dnsConfig, pod.Spec.DNSConfig)
	}

	return c.formDNSConfigFitsLimits(dnsConfig, pod), nil
}
```

### 如何在容器里配置 dns config / resolv.conf 的? 

kubelet 是通过 cri 来管理容器的, 在创建容器时是可以指定 dns 地址, dns-search 和 dns-opt 选项的, 实例化容器时会把配置写到容器内的 `/etc/resolv.conf` 文件里.

```c
--dns	The IP address of a DNS server. To specify multiple DNS servers, use multiple --dns flags. If the container cannot reach any of the IP addresses you specify, Google’s public DNS server 8.8.8.8 is added, so that your container can resolve internet domains.
--dns-search	A DNS search domain to search non-fully-qualified hostnames. To specify multiple DNS search prefixes, use multiple --dns-search flags.
--dns-opt	A key-value pair representing a DNS option and its value. See your operating system’s documentation for resolv.conf for valid options.
```

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301031512669.png)

### 总结

kubelet 根据 pod 的 dns 策略使用不同的 dns 配置, 然后调用 kubelet cri 向 dockerd 创建含有 dns 配置的容器.