# 源码分析 kubernetes nodeipam controller cidr 地址分配的实现原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301191910280.png)

> 基于 kubernetes v1.27.0 版本对 nodeipam controller 源码分析

在 k8s 的 `kube-controller-manager` 中 `nodeipam controller` 是用来为 node 节点分配可用的 ip cidr 地址段的控制器, 每个 node 拿到的 cidr 地址范围不会冲突的. 在为 node 分配地址段后, k8s kubelet 根据 node 关联的 cidr 地址段来为 pod 添加 ip 地址, 这里的又涉及到 cni 网络插件的调用, 后面再做分析.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301191644957.png)

这里分析下 kubernetes nodeipam controller 控制器的流程原理. nodeipam controller 控制器启动时通过 `--cluster-cidr`, `--node-cidr-mask-size` 和 `--cidr-allocator-type` 等参数生成 cidr 地址段生成器. 控制器内部又会启动 node informer 来监听 node 对象资源变化, 当触发 node 新增事件时, 为 node 申请分配 cidr 地址段, 然后向 apiserver 请求更新 node podcidrs 配置信息, 当触发删除事件时, 则释放这些 ip cidr 地址段.

nodeipam 只负责 pod cidr 地址, 至于 service clusterIP cidr 是在 apiserver 负责的.

## 实例化入口

实例化 `nodeIpamController` 控制器, allocatorType 分配器类型通过 `--cidr-allocator-type` 参数指定, 其默认值为 `RangeAllocator`.

```go
func NewNodeIpamController(
	...) (*Controller, error) {

	// 判空
	if kubeClient == nil {
		klog.Fatalf("kubeClient is nil when starting Controller")
	}

	// cloud 分配模式必须首先需要确保 cidr 段不为空, 然后判断 masksize 是否合法.
	if allocatorType != ipam.CloudAllocatorType {
		if len(clusterCIDRs) == 0 {
			klog.Fatal("Controller: Must specify --cluster-cidr if --allocate-node-cidrs is set")
		}

		for idx, cidr := range clusterCIDRs {
			mask := cidr.Mask
			if maskSize, _ := mask.Size(); maskSize > nodeCIDRMaskSizes[idx] {
				...
			}
		}
	}

	// 实例化 ipam controller 控制器对象
	ic := &Controller{
		cloud:                cloud,
		kubeClient:           kubeClient,
		eventBroadcaster:     record.NewBroadcaster(),
		lookupIP:             net.LookupIP,
		clusterCIDRs:         clusterCIDRs,
		serviceCIDR:          serviceCIDR,
		secondaryServiceCIDR: secondaryServiceCIDR,
		allocatorType:        allocatorType,
	}

	// 根据 allocator 的 type 实例化不同的 cidr allocator 分配器
	if ic.allocatorType == ipam.IPAMFromClusterAllocatorType || ic.allocatorType == ipam.IPAMFromCloudAllocatorType {
		ic.legacyIPAM = createLegacyIPAM(ic, nodeInformer, cloud, kubeClient, clusterCIDRs, serviceCIDR, nodeCIDRMaskSizes)
	} else {
		// 构建 ip 分配器的参数
		allocatorParams := ipam.CIDRAllocatorParams{
			ClusterCIDRs:         clusterCIDRs,
			ServiceCIDR:          ic.serviceCIDR,
			SecondaryServiceCIDR: ic.secondaryServiceCIDR,
			NodeCIDRMaskSizes:    nodeCIDRMaskSizes,
		}

		// 实例化 ip cidr 分配器
		ic.cidrAllocator, err = ipam.New(kubeClient, cloud, nodeInformer, clusterCIDRInformer, ic.allocatorType, allocatorParams)
		if err != nil {
			return nil, err
		}
	}

	// 赋值 node informer lister
	ic.nodeLister = nodeInformer.Lister()
	ic.nodeInformerSynced = nodeInformer.Informer().HasSynced

	return ic, nil
}
```

## 启动入口

启动 ipam 控制器, 内部根据分配类型启动不同的 cidr 分配器, 本文暂不分析 cloud 相关的分配器.

```go
func (nc *Controller) Run(stopCh <-chan struct{}) {
	klog.Infof("Starting ipam controller")
	defer klog.Infof("Shutting down ipam controller")

	// 等待 nodeinfomrer 数据同步到本地
	if !cache.WaitForNamedCacheSync("node", stopCh, nc.nodeInformerSynced) {
		return
	}

	// 根据分配类型启动不同的分配器.
	if nc.allocatorType == ipam.IPAMFromClusterAllocatorType || nc.allocatorType == ipam.IPAMFromCloudAllocatorType {
		go nc.legacyIPAM.Run(stopCh)
	} else {
		// 启动 cidr 分配器
		go nc.cidrAllocator.Run(stopCh)
	}

	<-stopCh
}
```

## cidr allocator 分配器的原理

### 创建 CIDRAllocator 实例的入口

下面为 cidrAllocator 实例的创建入口, 篇幅原因只分析 `RangeAllocator` 类型.

代码位置: `pkg/controller/nodeipam/ipam/cidr_allocator.go`

```go
func New(...) (CIDRAllocator, error) {
	// 获取所有的 node 列表
	nodeList, err := listNodes(kubeClient)
	if err != nil {
		return nil, err
	}

	// 根据 allocator 类型实例化 cidr 分配器
	switch allocatorType {
	case RangeAllocatorType:
		// RangeAllocator 为默认分配类型.
		return NewCIDRRangeAllocator(kubeClient, nodeInformer, allocatorParams, nodeList)

	case MultiCIDRRangeAllocatorType:
		return NewMultiCIDRRangeAllocator(kubeClient, nodeInformer, clusterCIDRInformer, allocatorParams, nodeList, nil)

	case CloudAllocatorType:
		return NewCloudCIDRAllocator(kubeClient, cloud, nodeInformer)

	default:
		return nil, fmt.Errorf("invalid CIDR allocator type: %v", allocatorType)
	}
}
```

### rangeAllocator 的实现原理

代码位置: `pkg/controller/nodeipam/ipam/range_allocator.go`

```go
func NewCIDRRangeAllocator(client clientset.Interface, nodeInformer informers.NodeInformer, allocatorParams CIDRAllocatorParams, nodeList *v1.NodeList) (CIDRAllocator, error) {
	// 遍历配置中的 cidrs 列表, 生成 cidrset 对象放到集合中.
	cidrSets := make([]*cidrset.CidrSet, len(allocatorParams.ClusterCIDRs))
	for idx, cidr := range allocatorParams.ClusterCIDRs { 
		// cidr 只有 ip 和子网, 通过参数 cidr 生成 cidr set 对象.
		cidrSet, err := cidrset.NewCIDRSet(cidr, allocatorParams.NodeCIDRMaskSizes[idx])
		if err != nil {
			return nil, err
		}
		cidrSets[idx] = cidrSet
	}

	ra := &rangeAllocator{
		// kube client
		client:                client,
		// 参数信息
		clusterCIDRs:          allocatorParams.ClusterCIDRs,
		// cidr 配置集合
		cidrSets:              cidrSets,
		// node lister
		nodeLister:            nodeInformer.Lister(),
		// 异步通知给 worker 协程, 请求 apiserver 更新 node 对象
		nodeCIDRUpdateChannel: make(chan nodeReservedCIDRs, cidrUpdateQueueSize),
		// 用来处理并发
		nodesInProcessing:     sets.NewString(),
	}

	// 如果有配置 service 专用的 cidr 段, 则进行配置.
	if allocatorParams.ServiceCIDR != nil {
		ra.filterOutServiceRange(allocatorParams.ServiceCIDR)
	} else {
		klog.V(0).Info("No Service CIDR provided. Skipping filtering out service addresses.")
	}

	if nodeList != nil {
		// 遍历当前的所有 node 对象.
		for _, node := range nodeList.Items {
			// 如果 node podcidrs 为空, 说明没有配置过可直接忽略.
			if len(node.Spec.PodCIDRs) == 0 {
				continue
			}

			// 如果当前 node 有分配过 pods cidrs 范围,  则需要占位处理.
			if err := ra.occupyCIDRs(&node); err != nil {
				return nil, err
			}
		}
	}

	// nodeinformer 注册 eventHandler 事件方法.
	nodeInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		// 当有新 node 上来时, 进行分配
		AddFunc: controllerutil.CreateAddNodeHandler(ra.AllocateOrOccupyCIDR),

		// 当 node 对象发生变更时, 且 PodCIDRs 段为空时才分配 cidr 段.
		UpdateFunc: controllerutil.CreateUpdateNodeHandler(func(_, newNode *v1.Node) error {
			if len(newNode.Spec.PodCIDRs) == 0 {
				return ra.AllocateOrOccupyCIDR(newNode)
			}
			return nil
		}),

		// 当 node 被清理时, 尝试释放其关联的 pod cidrs.
		DeleteFunc: controllerutil.CreateDeleteNodeHandler(ra.ReleaseCIDR),
	})

	return ra, nil
}
```

启动 `rangeAllocator` 分配器, 内部启动 30 个协程 worker 去处理 node cidr 的信息更新. worker 内部从 `nodeCIDRUpdateChannel` 管道消费事件, 并调用 `updateCIDRsAllocation` 向 apiserver 更新 node cidr 地址段的关系.

```go
const cidrUpdateWorkers = 30

func (r *rangeAllocator) Run(stopCh <-chan struct{}) {
	klog.Infof("Starting range CIDR allocator")
	defer klog.Infof("Shutting down range CIDR allocator")

	if !cache.WaitForNamedCacheSync("cidrallocator", stopCh, r.nodesSynced) {
		return
	}

	// 启动 30 个协程处理 worker 方法.
	for i := 0; i < cidrUpdateWorkers; i++ {
		go r.worker(stopCh)
	}

	<-stopCh
}

func (r *rangeAllocator) worker(stopChan <-chan struct{}) {
	for {
		select {
		case workItem, ok := <-r.nodeCIDRUpdateChannel:
			if !ok {
				return
			}

			// 已经为 node 分配 cidr 地址段, 调用方法向 apiserver 更新 node 对象.
			if err := r.updateCIDRsAllocation(workItem); err != nil {
				// 如果更新发生失败, 则重新入队
				r.nodeCIDRUpdateChannel <- workItem
			}
		case <-stopChan:
			return
		}
	}
}
```

`updateCIDRsAllocation` 方法把更新的 node spec.PodCIDRs 信息更新到 apiserver 上.

```go
func (r *rangeAllocator) updateCIDRsAllocation(data nodeReservedCIDRs) error {
	// 如果 node 的 PodCIDRs 跟分配的指标一样, 则直接跳出
	if len(node.Spec.PodCIDRs) == len(data.allocatedCIDRs) {
		match := true
		for idx, cidr := range cidrsString {
			if node.Spec.PodCIDRs[idx] != cidr {
				match = false
				break
			}
		}
		if match {
			klog.V(4).Infof("Node %v already has allocated CIDR %v. It matches the proposed one.", node.Name, data.allocatedCIDRs)
			return nil
		}
	}

	// 如果 node 的 cidr 不为空, 则需要在地址生成器里清理相关的 cidr 地址段.
	if len(node.Spec.PodCIDRs) != 0 {
		for idx, cidr := range data.allocatedCIDRs {
			// 尝试清理
			if releaseErr := r.cidrSets[idx].Release(cidr); releaseErr != nil {
				klog.Errorf("Error when releasing CIDR idx:%v value: %v err:%v", idx, cidr, releaseErr)
			}
		}
		return nil
	}

	// 尝试向 apiserver 更新 node cidrs 信息.
	for i := 0; i < cidrUpdateRetries; i++ {
		if err = nodeutil.PatchNodeCIDRs(r.client, types.NodeName(node.Name), cidrsString); err == nil {
			return nil
		}
	}

	// 当请求访问 apiserver 发生超时时, 主动释放下 cidr 地址段.
	if !apierrors.IsServerTimeout(err) {
		for idx, cidr := range data.allocatedCIDRs {
			if releaseErr := r.cidrSets[idx].Release(cidr); releaseErr != nil {
				klog.Errorf("Error releasing allocated CIDR for node %v: %v", node.Name, releaseErr)
			}
		}
	}

	// 如果 apiserver 更新 node cidrs 失败时, 则返回错误, 由调用方 worker 重写入队.
	// 不会丢失
	return err
}
```

### nodeinformer eventHandler 方法的实现

`AllocateOrOccupyCIDR` 方法用来为 node 新申请分配和占位 cidr.

```go
func (r *rangeAllocator) AllocateOrOccupyCIDR(node *v1.Node) error {
	// 防御式编程
	if node == nil {
		return nil
	}
	// 规避并发问题, 使用 nodesInProcessing set 集合避免并发处理同一个 node 对象.
	if !r.insertNodeToProcessing(node.Name) {
		return nil
	}

	// 如果 node 对象中 PodCIDRs 不为空, 则对 node 所属的 cidrs ip范围进行关联占用.
	if len(node.Spec.PodCIDRs) > 0 {
		return r.occupyCIDRs(node)
	}

	allocated := nodeReservedCIDRs{
		nodeName:       node.Name,
		allocatedCIDRs: make([]*net.IPNet, len(r.cidrSets)),
	}

	for idx := range r.cidrSets {
		// 从对应的 cidr 集合里分配
		podCIDR, err := r.cidrSets[idx].AllocateNext()
		if err != nil {
			// 如果有异常则在 nodesInProcessing 集合中删除
			r.removeNodeFromProcessing(node.Name)
			return fmt.Errorf("failed to allocate cidr from cluster cidr at idx:%v: %v", idx, err)
		}
		allocated.allocatedCIDRs[idx] = podCIDR
	}

	// 上面的逻辑只是为 node 申请分配 cidr 地址段
	// 而具体更新 node 对象信息则是异步由 worker 协程处理.
	r.nodeCIDRUpdateChannel <- allocated
	return nil
}
```

`ReleaseCIDR` 方法是用来释放 cidr 地址段, 在 ipam controller 里只有 node 资源对象被清理后, 才会调用 `ReleaseCIDR` 来释放 cidr 地址段.

```go
func (r *rangeAllocator) ReleaseCIDR(node *v1.Node) error {
	// 当 podCIDRs 为空时, 说明已经被释放过了.
	if node == nil || len(node.Spec.PodCIDRs) == 0 {
		return nil
	}

	// 遍历当前 node 的 podCIDRS 配置项.
	for idx, cidr := range node.Spec.PodCIDRs {
		// 把 cidr stirng 格式转换为 *net.IPNet
		_, podCIDR, err := netutils.ParseCIDRSloppy(cidr)
		if err != nil {
			return fmt.Errorf("failed to parse CIDR %s on Node %v: %v", cidr, node.Name, err)
		}

		...

		// 释放 cidr 地址段
		if err = r.cidrSets[idx].Release(podCIDR); err != nil {
			return fmt.Errorf("error when releasing CIDR %v: %v", cidr, err)
		}
	}
	return nil
}
```

### cidrset 的设计实现原理

`CidrSet` 是用来实现申请分配新的 cidr, 释放老的 cidr 和占位已存在 cidr 地址段的类.

源码位置: `pkg/controller/nodeipam/ipam/cidrset/cidr_set.go`

#### 实例化 cidrset 地址计算器

`NewCIDRSet` 里需要使用 `getMaxCIDRs` 计算出 clusterCIDR 地址段按照 `subNetMaskSize` 切分, 可以给 node 分配 cidr 的个数.

```go
func NewCIDRSet(clusterCIDR *net.IPNet, subNetMaskSize int) (*CidrSet, error) {
	clusterMask := clusterCIDR.Mask

	// 获取 clusterCIDR 的子网掩码
	clusterMaskSize, bits := clusterMask.Size()

	// 地址为空或者掩码相减大于 16, 子网配置不合理, 大家可以算下.
	if (clusterCIDR.IP.To4() == nil) && (subNetMaskSize-clusterMaskSize > clusterSubnetMaxDiff) {
		return nil, ErrCIDRSetSubNetTooBig
	}

	...

	// 计算出 cidr 地址段的个数
	maxCIDRs := getMaxCIDRs(subNetMaskSize, clusterMaskSize)
	cidrSet := &CidrSet{
		clusterCIDR:     clusterCIDR,
		nodeMask:        net.CIDRMask(subNetMaskSize, bits),
		clusterMaskSize: clusterMaskSize,
		maxCIDRs:        maxCIDRs,
		nodeMaskSize:    subNetMaskSize,
		label:           clusterCIDR.String(),
	}

	return cidrSet, nil
}
```

通过下面的例子分析下 `getMaxCIDRs` 过程. 比如 clusetrCIDR 参数为 `10.0.0.0/16`, 主机子网大小为 `24`, 那么通过 `getMaxCIDRs(24, 16)` 可以拿到的 256 个 cidr 配置段 (0-255), 每个 cidr 可以分配 254 个地址 (0 为网络号, 255 为广播地址). k8s node 默认下可以开 110 个 pod, 这个 cidr 地址是足够了.

```go
package main

import (
	"fmt"

	netutils "k8s.io/utils/net"
)

func main() {
	cidr := "10.0.0.0/16"
	_, clusterCIDR, _ := netutils.ParseCIDRSloppy(cidr)
	fmt.Println("clusterCIDR:", clusterCIDR.String()) // output: 10.0.0.0/16

	clusterMaskSize, _ := clusterCIDR.Mask.Size()
	fmt.Println("clusterMaskSize: ", clusterMaskSize) // output: 16

	max := getMaxCIDRs(24, clusterMaskSize)
	fmt.Println("getMaxCIDRs:", max) // output: 256
}

func getMaxCIDRs(subNetMaskSize, clusterMaskSize int) int {
	return 1 << uint32(subNetMaskSize-clusterMaskSize)
}
```

ipam 代码中有不少子网掩码换算和子网划分等地址换算的调用, 这需要一定的网络基础知识才方便理解.

####  申请 cidr 地址段 (AllocateNext)

`AllocateNext` 方法用来申请分配可用的 cidr 地址段. `used` 类型是 `bit.Int`, 其目的通过 bitmap 结构来标记 cidr 是否被分配.

比如通过 getMaxCIDRs 得出当前最多可以分配 256 个 cidr 地址段, 把已经分配过的 cidr 段在 used bitmap 对应位置标记为 1, 没被分配过的 cidr 标记为 0.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301191554583.png)

代码位置: `pkg/controller/nodeipam/ipam/cidrset/cidr_set.go`

```go
func (s *CidrSet) AllocateNext() (*net.IPNet, error) {
	s.Lock()
	defer s.Unlock()

	// 如果当前分配的 cidr 已经等于 maxCIDRs, 则直接无剩余 cidr 地址段错误.
	if s.allocatedCIDRs == s.maxCIDRs {
		return nil, ErrCIDRRangeNoCIDRsRemaining
	}

	candidate := s.nextCandidate
	var i int
	// 从上次的 candidate 开始循环遍历, 直到找到可用的 cidr.
	for i = 0; i < s.maxCIDRs; i++ {
		// 如果当前位还未使用, 则使用该 candidate.
		if s.used.Bit(candidate) == 0 {
			break
		}

		// 跟 maxCIDRs 取摸计算出 candidate
		candidate = (candidate + 1) % s.maxCIDRs
	}

	// 下次的 candidate 位置
	s.nextCandidate = (candidate + 1) % s.maxCIDRs

	// 配置 used bitmap 中 candidate 位置 bit 为 1
	s.used.SetBit(&s.used, candidate, 1)

	// 每次分配成功都会加一
	s.allocatedCIDRs++

	return s.indexToCIDRBlock(candidate), nil
}
```

`indexToCIDRBlock` 方法可通过 index 位置进行位运算后获取 cidr 地址段.

```go
func (s *CidrSet) indexToCIDRBlock(index int) *net.IPNet {
	var ip []byte
	switch {
	case s.clusterCIDR.IP.To4() != nil:
		{
			j := uint32(index) << uint32(32-s.nodeMaskSize)
			ipInt := (binary.BigEndian.Uint32(s.clusterCIDR.IP)) | j
			ip = make([]byte, net.IPv4len)
			binary.BigEndian.PutUint32(ip, ipInt)
		}
	case s.clusterCIDR.IP.To16() != nil:
		...
	}

	return &net.IPNet{
		IP:   ip,
		Mask: s.nodeMask,
	}
}
```

通过单元测试分析下 `indexToCIDRBlock` 执行的过程. 比如 ipam 可分配的地址段为 `127.123.3.0/16`, 分给 node 的子网为 `24`, 那么可以分配的 cidr 地址范围是 `127.123.0.0 - 127.123.255.0`. 当 index 为 0 时, cidr 为 `127.123.0.0`, index 为 10 时, cidr 为 `127.123.10.0`.

```go
func TestIndexToCIDRBlock(t *testing.T) {
	cases := []struct {
		clusterCIDRStr string
		subnetMaskSize int
		index          int
		CIDRBlock      string
		description    string
	}{
		{
			clusterCIDRStr: "127.123.3.0/16",
			subnetMaskSize: 24,
			index:          0,
			CIDRBlock:      "127.123.0.0/24",
			description:    "1st IP address indexed with IPv4",
		},
		{
			clusterCIDRStr: "127.123.0.0/16",
			subnetMaskSize: 24,
			index:          15,
			CIDRBlock:      "127.123.15.0/24",
			description:    "16th IP address indexed with IPv4",
		},
		{
			clusterCIDRStr: "192.168.5.219/28",
			subnetMaskSize: 32,
			index:          5,
			CIDRBlock:      "192.168.5.213/32",
			description:    "5th IP address indexed with IPv4",
		},
	}
	for _, tc := range cases {
		_, clusterCIDR, _ := netutils.ParseCIDRSloppy(tc.clusterCIDRStr)
		a, err := NewCIDRSet(clusterCIDR, tc.subnetMaskSize)
		if err != nil {
			t.Fatalf("error for %v ", tc.description)
		}
		cidr := a.indexToCIDRBlock(tc.index)
		if cidr.String() != tc.CIDRBlock {
			t.Fatalf("error for %v index %d %s", tc.description, tc.index, cidr.String())
		}
	}
}
```

#### 释放不用的 cidr 地址段

`Release` 方法用做在地址分配器里释放对应的 cidr 地址段, 标记为空闲 ( 0 ).

```go
// Release releases the given CIDR range.
func (s *CidrSet) Release(cidr *net.IPNet) error {
	// 通过 cidr 获取 begin 和 end
	begin, end, err := s.getBeginningAndEndIndices(cidr)
	if err != nil {
		return err
	}

	s.Lock()
	defer s.Unlock()

	// 从 begin - end 进行遍历, 把对应的的位配置为空位, 0 为未使用.
	for i := begin; i <= end; i++ {
		if s.used.Bit(i) != 0 {
			s.used.SetBit(&s.used, i, 0)
			s.allocatedCIDRs--
		}
	}

	return nil
}
```

`getBeginningAndEndIndices` 方法用来获取 cidr 在 used bitmap 中的索引偏移量. 函数实现原理还是颇为繁杂, 其内部有不少的位运算, 这里就分析地址的换算原理.

#### 占位已被使用 CIDR 地址段

`Occupy` 主要是在 ipam 里占位已经在 node 里配置过 cidr 的地址. 分配器在启动时会遍历集群中 node 对象列表, 把这些 node 的 cidr 都在 ipam 中占住, 避免被新的 node 分配占用.

`Occupy` 方法跟 `Release` 相反, Release 是把指定的 cidr 给释放掉, 而 `Occupy` 是把指定的 cidr 给占位, 只是在 cidr 分配器里标记该 cidr 段已经被分配, 无需向 apiserver 请求更新.

```go
func (s *CidrSet) Occupy(cidr *net.IPNet) (err error) {
	// 通过 cidr 获取 index 的 begin 和 end
	begin, end, err := s.getBeginningAndEndIndices(cidr)
	if err != nil {
		return err
	}

	s.Lock()
	defer s.Unlock()
	for i := begin; i <= end; i++ {
		if s.used.Bit(i) == 0 {
			// 设置该位 bit 为 1, 也就是已被使用.
			s.used.SetBit(&s.used, i, 1)
			s.allocatedCIDRs++
		}
	}

	return nil
}
```

## 总结

nodeIpamController 的原理就是这样的. 控制器启动时通过 `--cluster-cidr` 和 `--cidr-allocator-type` 生成 cidr 地址段生成器. 其内部启动 nodeInformer 监听 node 对象资源, 当触发 node 新增事件时, 为 node 申请分配 cidr 地址段, 然后向 apiserver 请求更新 node podcidrs 配置信息, 当触发删除事件时, 则释放这些 ip cidr 地址段.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301191644957.png)

pod 的 ip 地址是依赖 nodeipma 分配的, 而 service 的 clusetrIP 则是 kube apiserver 来实现的, 其 clusterIP 集群地址范围是由参数 `--service-cluster-ip-range` 控制的, 默认是 `10.0.0.0/24` 地址范围, 当然是不足够的了, 通常会把子网调整到 `16`. service 和 pod 的 cidr 范围尽量不要冲突, ipma controller 启动时会读取 `--service-cluster-ip-range` 参数作为 service 的 cidr 地址段, 然后在分配器调用 `Occupy` 占位 service 地址范围以规避分配冲突.