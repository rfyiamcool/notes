## benchmark istio

### 环境

硬件资源

- 三台阿里云主机
- Intel(R) Xeon(R) Gold 6161 CPU @ 2.20GHz
- 8u 32g

网络状态

- 1000mb
- 三台主机之间的ping时延在 0.2 ms;

测试条件

- k8s version = v1.20.6
- istio version = 1.10.2

![](https://xiaorui.cc/image/2020/%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C.jpg)

### tool

推荐用 wrk, wrk2 或 hey 进行压测，wrk用来测试最大QPS, wrk2用来测试恒定输出下的各个指标，也可以选择 vegeta 或 ab.

需要注意的是 wrk, hey 默认为长连接，而 ab 默认短连接, 可通过 -k 改为长连接, ab 还有一个缺点是单线程.

- [https://github.com/wg/wrk](https://github.com/wg/wrk)
- [https://github.com/giltene/wrk2](https://github.com/giltene/wrk2)
- [https://github.com/rakyll/hey](https://github.com/rakyll/hey)
- [https://github.com/tsenart/vegeta](https://github.com/tsenart/vegeta)

### k8s yaml

直接 kubectl apply k8s.yaml 就可以了.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wrk
  labels:
    app: wrk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wrk
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "true"
      labels:
        app: wrk
    spec:
      nodeSelector:
        kubernetes.io/hostname: service-mesh-0001
      containers:
      - name: wrk
        image: bootjp/wrk2
        command: ['sh', '-c', 'echo nginx is running! && sleep 8640000']
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: wrk
  labels:
    app: wrk
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: wrk

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-wrk
  labels:
    app: nginx-wrk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-wrk
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "true"
      labels:
        app: nginx-wrk
    spec:
      nodeSelector:
        kubernetes.io/hostname: service-mesh-0002
      containers:
      - name: debug-tool
        image: busybox
        command: ['sh', '-c', 'echo debug-tool  is running! && sleep 8640000']
        imagePullPolicy: IfNotPresent #Always

      - name: nginx-wrk
        image: nginx:latest
        command: ['sh', '-c', 'echo nginx is running! && nginx']
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-wrk
  labels:
    app: nginx-wrk
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-wrk

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-proxy
  labels:
    app: nginx-proxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-proxy
  template:
    metadata:
      #annotations:
      #  sidecar.istio.io/inject: "true"
      labels:
        app: nginx-proxy
    spec:
      nodeSelector:
        kubernetes.io/hostname: service-mesh-0003
      containers:
      - name: debug-tool
        image: busybox
        command: ['sh', '-c', 'echo debug-tool  is running! && sleep 8640000']
        imagePullPolicy: IfNotPresent #Always

      - name: nginx-proxy
        image: nginx:latest
        command: ['sh', '-c', 'echo nginx is running! && sleep 8640000'] # setting nginx proxy
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 80
        - containerPort: 8181

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-proxy
  labels:
    app: nginx-proxy
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-proxy
```

### latency benchmark

压测预设恒定输出下的吞吐和时延, 使用压测工具 wrk2. 每种测试条件差不多进行5次，最后取平均数. rate是说明wrk2重试每秒发送rate数量的请求, 实际的qps跟rate相差一点数值, 很小.

#### with sidecar

| mode    | connection | rate | duration | avg-latency | max-latency |
|---------|------------|------|----------|-------------|-------------|
| sidecar | 10         | 100  | 10s      |   1.59ms    |  3.05ms     |
| sidecar | 50         | 500  | 10s      |   1.80ms    |  6.31ms     |
| sidecar | 100        | 1000 | 10s      |   2.11ms    |  9.94ms     |
| sidecar | 150        | 2000 | 10s      |   3.30ms    | 16.37ms     |
| sidecar | 200        | 5000 | 10s      |   4.66ms    | 30.74ms     |

#### without sidecar

| mode    | connection | rate | duration | avg-latency | max-latency |
|---------|------------|------|----------|-------------|-------------|
| no      | 10         | 100  | 10s      |   0.95ms    | 1.98ms      |
| no      | 50         | 500  | 10s      |   0.98ms    | 2.76ms      |
| no      | 100        | 1000 | 10s      |   1.01ms    | 4.76ms      |
| no      | 150        | 3000 | 10s      |   1.10ms    | 6.98ms      |
| no      | 200        | 5000 | 10s      |   1.11ms    | 7.93ms      |

### max benchmark

使用原版 wrk 对服务进行暴力压测, 得出可以正常输出的最大 QPS, 另外得出在压测下的时延表现.

#### benchmark with sidecar

```bash
# wrk -c 2000 -d 10s http://nginx-wrk/wrk
Running 10s test @ http://nginx-wrk/wrk
  2 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   197.30ms   21.60ms 464.13ms   83.42%
    Req/Sec     5.10k     1.79k    8.79k    65.48%
  100542 requests in 10.05s, 36.84MB read
  Non-2xx or 3xx responses: 100542
Requests/sec:  10003.13
Transfer/sec:      3.66MB
```

#### benchmark without sidecar

```bash
# wrk -c 2000 -d 10s http://nginx-wrk/wrk

Running 10s test @ http://nginx-wrk/wrk
  2 threads and 2000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    10.18ms    8.71ms 215.40ms   85.86%
    Req/Sec    72.53k     3.74k   80.96k    69.54%
  1446399 requests in 10.09s, 424.85MB read
  Non-2xx or 3xx responses: 1446399
Requests/sec: 143318.73
Transfer/sec:     42.10MB
```

#### max benchmark report

非 sidecar 可以干到 14w qps, 而 sidecar 只有 10000 左右. 在 sidecar 模式下，不管你怎么调压测参数，qps都保持在 1w 左右, envoy 的进程在压力请求下保持在满负载的 200% 左右.

### qps为什么无法提升?

为什么性能差距这么明显 ??? 😅 为什么 envoy 的 cpu 开销最高只能 `200%` .

![](https://xiaorui.cc/image/2020/20210716171014.png)

通过 kubectl describe pod 查看了业务相关的sidecar, cpu limit 只有2U，内存也仅仅有1GB.

```yaml
  istio-proxy:
    Container ID:  docker://9283ff21619fa376b05c53bc920c3c3c0f9918c73e9ccd06fb039fe1b268d1f6
    Image:         docker.io/istio/proxyv2:1.10.2
    Image ID:      docker-pullable://istio/proxyv2@sha256:ffbc024407c89d15b102242b7b52c589ff3618c5117585007abbdf03da7a6528
    Port:          15090/TCP
    Host Port:     0/TCP
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      --serviceCluster
      nginx-wrk.$(POD_NAMESPACE)
      --proxyLogLevel=warning
      --proxyComponentLogLevel=misc:error
      --log_output_level=debug
      --concurrency
      2
    State:          Running
      Started:      Thu, 15 Jul 2021 18:24:30 +0800
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     2
      memory:  1Gi
    Requests:
      cpu:      100m
      memory:   128Mi
```

其中 `concurrency` 参数为 envoy 的线程数, 可以理解为 eventloop事件循环, 该参数需要跟 linux.cpu 对齐.

### 结论

- istio-proxy 容器自身是有cpu/mem资源限制, 默认为 `2cpu 1G`;
- 正常的并发请求下每层 sidecar 会增加时延 `0.5ms` 左右;
- 使用默认istio-proxy的资源配置下, 最多可以抗住 `1w` 左右的HTTP请求并发.
- 建议根据业务请求量级来调整 sidecar 资源.