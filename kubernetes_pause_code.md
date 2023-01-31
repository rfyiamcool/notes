# 源码分析 kubernetes pause 容器的设计实现原理

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301311800927.png)

## k8s pause 容器干嘛的

已知 Pod 是 k8s 里最小调度单元, 一个 Pod 可以由一组容器组成的. 那么这些容器之间共享存储和网络资源 ?

docker 启动容器时是可以做到让不同容器之间的 namespace 共享, 共享原则必然是 A 共享 B 的 namespace, 或者翻着来. 总之必然是需要一个共享和被共享的关系. 那么一个 pod 里有两个业务容器, 谁共享谁的呢 ? 我们知道业务容器有不稳定的可能, 比如如果 B 容器 A 的网络命名空间 ( net namespace ), 这时候 A 挂了被 docker 重建新容器, 那么这时候 A 和 B 就无法做到共享了.

**一句话解释, 使用业务容器做共享基础容器不安全.** 

#### 使用 pause 实现容器间的 namespace 共享?

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301311806020.png)

k8s 搞了一个 pause 容器, 又叫 infra container 基础容器. 所有的业务容器都要依赖于 pause 容器，因此在 Pod 启动时, 它总是创建的第一个容器, 可以说 Pod 的生命周期就是 `pause` 容器的生命周期.

pause 容器的景象大小在 500 KB 左右, 特别的轻便, 另外该容器绝大部分时间都陷入等待, 可以说没有性能开销. 网络 IP 地址及路由的相关配置其实都是在 pause 容器内完成的.

kubelet 有个 `--pod-infra-container-image` 参数就是控制 pause 的容器镜像地址. 

```
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.5
```

#### k8s pod 开启了哪些 namespace 共享

pod 默认开启共享了 NETWORK, IPC 和 UTS 命名空间. 其他命名空间 namespace 需要在 pod 配置才可开启. 比如可以通过 `shareProcessNamespace = true` 开启 PID 命名空间的共享, 共享 pid 命名空间后, 容器内可以互相查看彼此的进程.

## pause 容器共享 namespace 的实现原理

pause 容器用来实现其他容器之间的 namespace 共享, 下面手动操作下 pause 如何实现容器间的 namespace 共享. 

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301311552098.png)

1. 我们首先在节点上运行一个 pause 容器, 并且映射把本机的 80 端口映射到容器内的 80 端口. pause 容器 k8s 官方会推到 gcr.io, 因为国内的原因请使用云厂商同步打包的镜像.

```bash
[root@k8s-node1 ~]# docker run -d --name pause -p 80:80 registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6
```

2. 创建一个 nginx 容器, 挂在本地文件目录到容器内, 并监听 80 端口把数据转发到本地的 2368 端口上. nginx 容器共享使用 pause 容器的 net, ipc 和 pid 命名空间.

```
[root@k8s-node1 ~]# cat <<EOF >> nginx.conf
error_log stderr;
events { worker_connections  1024; }
http {
    access_log /dev/stdout combined;
    server {
        listen 80 default_server;
        server_name xiaorui.cc;
        location / {
            proxy_pass http://127.0.0.1:2368;
        }
    }
}
EOF

[root@k8s-node1 ~]# docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause nginx
```

3. 启动 ghost 博客系统, 共享使用 pause 容器的 net 和 ipc 命名空间.

```
[root@linux-node2 ~]# docker run -d --name ghost --net=container:pause --ipc=container:pause --pid=container:pause ghost
```

4. 在 ghost 容器内可以看到 puase 容器的 pause 主进程, nginx 容器内的 nginx master/worker 进程, 还有本容器内的 node web 进程. 需要注意的是 pause 容器的 PID 是 1.

```bash
[root@k8s-node1 ~]# docker exec -it ghost /bin/bash
root@xiaorui.cc:/var/lib/ghost# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   1012     4 ?        Ss   08:27   0:00 /pause
systemd+     9  0.0  0.0  32932  1812 ?        S    08:29   0:00 nginx: worker process
root         5  0.0  0.0  32472  3168 ?        Ss   08:29   0:00 nginx: master process nginx -g daemon off;
node        10  0.5  2.1 888112 38189 ?       Ssl  08:30   0:03 node current/index.js
root        87  0.0  0.0  17496  1148 pts/0    R+   08:41   0:00 ps aux
root        83  0.2  0.0  20240  1912 pts/0    Ss   08:41   0:00 /bin/bash
```

到此手动的完成了容器之间的 net, pid, ipc 的命名空间共享. 其实现原理简单明了. 就是创建一个 pause 作为基础容器, 然后业务容器在启动时共享 pause 容器的命名空间.

```
- --net=container:{name}
- --ipc=container:{name}
- --pid=container:{name}
```

## pause 源码解析

pause 容器主要做两件事情.

- 注册各种信号处理函数，主要处理两类信息：退出信号和 child 信号. 当收到 SIGINT 或是 SIGTERM 后, pause 进程可直接退出. 收到 SIGCHLD 信号, 则调用 waitpid 进行回收进程.
- 主进程 for 循环调用 pause() 函数，使进程陷入休眠状态, 不占用 cpu 资源, 直到被终止或是收到信号.

k8s pod 的 `shareProcessNamespace` 参数会使一个 pod 内的几个容器间共享 pid namespace, 这样容器间互相可以看到各自的进程. 当业务容器退出时, 说明容器的运行进程退出, 那么 pause 会收到 `SIGCHLD` 信号, pause 会调用 `waitpid` 进行收尾, 不让挂掉的进程产生僵尸状态.

k8s pause 的代码位置 `build/pause/linux/pause.c`

```c
...

/* SIGINT, SIGTERM 信号会调用该函数. */
static void sigdown(int signo) {
  psignal(signo, "Shutting down, got signal");
  /* sigint 和 sigterm 都是正常干掉进程, exit code 为 0. */
  exit(0);
}

/* SIGCHLD 信号会调用该函数. */
static void sigreap(int signo) {
  /* waitpid 监听进程组的子进程退出, WNOHANG 是非阻塞标记, 当没有找到子进程退出时, 不会阻塞. */
  /* -1 是什么？ 1 为 pod 主进程, 通常也是pgid, -1 则是指进程组 1 */
  while (waitpid(-1, NULL, WNOHANG) > 0)
    ;
}

int main(int argc, char **argv) {
  int i;
  for (i = 1; i < argc; ++i) {
    if (!strcasecmp(argv[i], "-v")) {
      /* 打印 paruse.c 版本 */
      printf("pause.c %s\n", VERSION_STRING(VERSION));
      return 0;
    }
  }

  if (getpid() != 1)
    /* 如果不是 1 号进程, 则打印错误. */
    fprintf(stderr, "Warning: pause should be the first process\n");

  /* 注册 signal 信号对应的回调方法 */
  if (sigaction(SIGINT, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 1;
  if (sigaction(SIGTERM, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 2;
  if (sigaction(SIGCHLD, &(struct sigaction){.sa_handler = sigreap,
                                             .sa_flags = SA_NOCLDSTOP},
                NULL) < 0)
    return 3;

  for (;;)
    pause(); // 等待 signal 信号
  fprintf(stderr, "Error: infinite loop terminated\n");
  return 42;
}
```

#### `pause.c` 代码中涉及到几个系统个调用.

**pause**

阻塞等待信号 signal. 

[https://linux.die.net/man/2/pause](https://linux.die.net/man/2/pause)

**waitpid**

等待子进程退出后回收进程的 `task_struct` 对象, 避免出现子进程已退出, 但通过 `ps aux` 出现僵尸进程状态. 

什么是僵尸进程 ? 简单说当子进程已经退出, 但因为其父进程没有回收释放, 导致仍然在进程表中的存在. 这里父进程需要调用 `waitpid` 系统调用来回收进行. 其实直接在 main 里配置忽略 `SIGCHLD` 信号也可以, 这样子进程的僵尸回收交给了 init 首进程, 也就是内核进程帮忙回收, 但不适合容器场景.

没学过 apue 进程方面同学, 可以学下 孤儿进程和僵尸进程是怎么一回事.

[https://linux.die.net/man/2/waitpid](https://linux.die.net/man/2/waitpid)

### kubelet pause 容器的操作

`SyncPod` 是 kubelet 组件创建 pod 的核心方法. 关于 kubelet 创建 pod 原理这里就不再复述, 有兴趣的同事看下以前写过的 kubelet 原理文章.

简化描述下 syncpod 的实现流程. 首先创建一个沙盒容器, 所谓的沙盒容器就是 pause 容器. 后面再分别启动临时容器, init 容器, 业务的容器. 在创建这些容器时, 需要传入 pause 容器的 id, 结合上面的 pause 容器可以分析出 docker 容器之前需要做 namespace 的共享.

源码位置: `pkg/kubelet/kuberuntime/kuberuntime_manager.go`

```go
// SyncPod syncs the running pod into the desired pod by executing following steps:
func (m *kubeGenericRuntimeManager) SyncPod(...) {
	...

	// 创建 sandbox 沙盒容器, 其实就是 pause 容器
	podSandboxID := podContainerChanges.SandboxID
	if podContainerChanges.CreateSandbox {
		podSandboxID, msg, err = m.createPodSandbox(ctx, pod, podContainerChanges.Attempt)
		...
	}

	podIP := ""
	if len(podIPs) != 0 {
		podIP = podIPs[0]
	}

	start := func(ctx context.Context, typeName, metricLabel string, spec *startSpec) error {
		...

		// 创建并启动容器, 需要把 sandbox 容器的 id 传入进去.
		if msg, err := m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs); err != nil {
			...
		}
		return nil
	}

	// 启动临时容器
	for _, idx := range podContainerChanges.EphemeralContainersToStart {
		start(ctx, "ephemeral container", metrics.EphemeralContainer, ephemeralContainerStartSpec(&pod.Spec.EphemeralContainers[idx]))
	}

	// 启动 init 容器
	if container := podContainerChanges.NextInitContainerToStart; container != nil {
		// Start the next init container.
		if err := start(ctx, "init container", metrics.InitContainer, containerStartSpec(container)); err != nil {
			return
		}
	}

	// 启动业务容器
	for _, idx := range podContainerChanges.ContainersToStart {
		start(ctx, "container", metrics.Container, containerStartSpec(&pod.Spec.Containers[idx]))
	}
}
```

当前 k8s 是通过 cri 运行时接口来调用 containerd 服务的, containerd 实现了 `CreateContainer` 创建容器的 grpc 接口.

源码位置: `pkg/kubelet/cri/remote/remote_runtime.go`

```go
func (r *remoteRuntimeService) CreateContainer(ctx context.Context, podSandBoxID string, config *runtimeapi.ContainerConfig, sandboxConfig *runtimeapi.PodSandboxConfig) (string, error) {
	ctx, cancel := context.WithTimeout(ctx, r.timeout)
	defer cancel()

	return r.createContainerV1(ctx, podSandBoxID, config, sandboxConfig)
}

func (r *remoteRuntimeService) createContainerV1(ctx context.Context, podSandBoxID string, config *runtimeapi.ContainerConfig, sandboxConfig *runtimeapi.PodSandboxConfig) (string, error) {
	resp, err := r.runtimeClient.CreateContainer(ctx, &runtimeapi.CreateContainerRequest{
		PodSandboxId:  podSandBoxID,
		Config:        config,
		SandboxConfig: sandboxConfig,
	})
	if err != nil {
		return "", err
	}

	klog.V(10).InfoS("[RemoteRuntimeService] CreateContainer", "podSandboxID", podSandBoxID, "containerID", resp.ContainerId)
	if resp.ContainerId == "" {
		errorMessage := fmt.Sprintf("ContainerId is not set for container %q", config.Metadata)
		err := errors.New(errorMessage)
		klog.ErrorS(err, "CreateContainer failed")
		return "", err
	}

	return resp.ContainerId, nil
}
```

关于 docker (containerd) 的具体实现, 这里就不做分析.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301312226802.png)

## 总结

pause 作为 infra 基础容器, 用来实现 pod 内多个容器之间的 namespace 共享.