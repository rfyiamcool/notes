# volcano 云原生批量计算平台的 controller 控制器设计实现

volcano 的原文地址在 [xiaorui.cc](xiaorui.cc), 后面对 volcano 的架构及技术实现原理会持续补充.

## volcano controller 的实现

- [controller 控制器实现](#volcano controller 的实现)
    - [QueueController 控制器](#QueueController 控制器)
        - [启动入口 Run](#启动入口 Run)
        - [processNextWorkItem](#processNextWorkItem)
        - [handleQueue](#handleQueue)
        - [state 状态处理](#state 状态处理)
        - [三种 queue 处理方法](#三种 queue 处理方法)
    - [JobController 控制器](#JobController 控制器)
        - [启动入口 Run](#启动入口 Run)
        - [worker.processNextReq](#worker.processNextReq)
        - [SyncJob](#SyncJob)
        - [KillJob](#KillJob)
        - [State](#State)
    - [PodGroupController 控制器](#PodGroupController 控制器)
        - [启动入口 Run](#启动入口 Run)
        - [processNextReq](#processNextReq)

### 如何启动 volcano controller 控制器

```go
// Run the controller.
func Run(opt *options.ServerOption) error {
    // 获取启动 controllers 的方法 
    run := startControllers(config, opt)

    // 创建选举客户端
    leaderElectionClient, err := kubeclientset.NewForConfig(rest.AddUserAgent(config, "leader-election"))
    if err != nil {
        return err
    }

    // ...

    // 创建选举锁对象
    rl, err := resourcelock.New(resourcelock.ConfigMapsResourceLock,
        opt.LockObjectNamespace,
        "vc-controller-manager",
        leaderElectionClient.CoreV1(),
        leaderElectionClient.CoordinationV1(),
        resourcelock.ResourceLockConfig{
            Identity:      id,
            EventRecorder: eventRecorder,
        })
    if err != nil {
        return fmt.Errorf("couldn't create resource lock: %v", err)
    }

    // 进行选举，如果拿到锁，则执行控制器逻辑，没拿到则等待。
    // 如果在拿到锁后，发生异常导致续锁失败，导致被抢占，则退出进程。
    leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
        Lock:          rl,
        LeaseDuration: leaseDuration,
        RenewDeadline: renewDeadline,
        RetryPeriod:   retryPeriod,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: run,
            OnStoppedLeading: func() {
                klog.Fatalf("leaderelection lost")
            },
        },
    })
    return fmt.Errorf("lost lease")
}

func startControllers(config *rest.Config, opt *options.ServerOption) func(ctx context.Context) {
    controllerOpt := &framework.ControllerOption{}

    controllerOpt.SchedulerName = opt.SchedulerName
    controllerOpt.WorkerNum = opt.WorkerThreads
    controllerOpt.MaxRequeueNum = opt.MaxRequeueNum

    controllerOpt.KubeClient = kubeclientset.NewForConfigOrDie(config)
    controllerOpt.VolcanoClient = vcclientset.NewForConfigOrDie(config)
    controllerOpt.SharedInformerFactory = informers.NewSharedInformerFactory(controllerOpt.KubeClient, 0)

    return func(ctx context.Context) {
        framework.ForeachController(func(c framework.Controller) {
            if err := c.Initialize(controllerOpt); err != nil {
                return
            }

            go c.Run(ctx.Done())
        })

        <-ctx.Done()
    }
}

// 启动时 queue、job、pg 会注册到这里。
var controllers = map[string]Controller{}

func ForeachController(fn func(controller Controller)) {
    // 遍历执行各个控制器。
    for _, ctrl := range controllers {
        fn(ctrl)
    }
}
```

### QueueController 控制器

#### 启动入口 Run

启动 queue、pg、cmd informer，在等待同步完成后，异步调用 worker 和 commandWorker 协程。

```go
func (c *queuecontroller) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()
    defer c.queue.ShutDown()
    defer c.commandQueue.ShutDown()

    klog.Infof("Starting queue controller.")
    defer klog.Infof("Shutting down queue controller.")

    go c.queueInformer.Informer().Run(stopCh)
    go c.pgInformer.Informer().Run(stopCh)
    go c.cmdInformer.Informer().Run(stopCh)

    if !cache.WaitForCacheSync(stopCh, c.queueSynced, c.pgSynced, c.cmdSynced) {
        klog.Errorf("unable to sync caches for queue controller.")
        return
    }

    go wait.Until(c.worker, 0, stopCh)
    go wait.Until(c.commandWorker, 0, stopCh)

    <-stopCh
}
```

#### processNextWorkItem

监听 queue 队列的数据，该队列的数据由 queueInformer eventHandler 来产生。

```go
func (c *queuecontroller) worker() {
    for c.processNextWorkItem() {
    }
}

func (c *queuecontroller) processNextWorkItem() bool {
    obj, shutdown := c.queue.Get()
    if shutdown {
        return false
    }
    defer c.queue.Done(obj)

    req, ok := obj.(*apis.Request)
    if !ok {
        klog.Errorf("%v is not a valid queue request struct.", obj)
        return true
    }
  
    // 这里是 handleQueue
    err := c.syncHandler(req)
    c.handleQueueErr(err, obj)

    return true
}
```

#### handleQueue

根据 queue 状态返回不同的状态处理方法，然后变更状态。

```go
func (c *queuecontroller) handleQueue(req *apis.Request) error {
    // 获取 queue 对象
    queue, err := c.queueLister.Get(req.QueueName)
    if err != nil {
        if apierrors.IsNotFound(err) {
            return nil
        }

        return fmt.Errorf("get queue %s failed for %v", req.QueueName, err)
    }

    // 根据 queue 状态返回不同的状态处理方法
    queueState := queuestate.NewState(queue)
    if queueState == nil {
        return fmt.Errorf("queue %s state %s is invalid", queue.Name, queue.Status.State)
    }

    // 处理状态，本质都是更新 queue 对象状态.
    if err := queueState.Execute(req.Action); err != nil {
        return err
    }

    return nil
}
```

#### state 状态处理

queue 有各种各样的状态处理方法，这里拿 openState 举例说明.

```go
type openState struct {
    queue *v1beta1.Queue
}

func (os *openState) Execute(action v1alpha1.Action) error {
    switch action {
    case v1alpha1.OpenQueueAction:
        // open 状态，进行 syncQueue 调和.
        return SyncQueue(os.queue, func(status *v1beta1.QueueStatus, podGroupList []string) {
            status.State = v1beta1.QueueStateOpen
        })
    case v1alpha1.CloseQueueAction:
        // close 状态，进行 queue 收尾操作.
        return CloseQueue(os.queue, func(status *v1beta1.QueueStatus, podGroupList []string) {
            if len(podGroupList) == 0 {
                status.State = v1beta1.QueueStateClosed
                return
            }
            status.State = v1beta1.QueueStateClosing
        })
    default:
        // 其他状态，则调用 syncQueue 调和.
        return SyncQueue(os.queue, func(status *v1beta1.QueueStatus, podGroupList []string) {
            specState := os.queue.Status.State
            if len(specState) == 0 || specState == v1beta1.QueueStateOpen {
                status.State = v1beta1.QueueStateOpen
                return
            }

            if specState == v1beta1.QueueStateClosed {
                if len(podGroupList) == 0 {
                    status.State = v1beta1.QueueStateClosed
                    return
                }
                status.State = v1beta1.QueueStateClosing

                return
            }

            status.State = v1beta1.QueueStateUnknown
        })
    }
}
```

#### 三种 queue 处理方法

##### syncQueue

- 获取 queue 对应的 podGroup 集合
- 根据各个 podGroup 的状态，累计 queue status 指标
- 更新 queue 对象

##### openQueue

- 从 client 获取最新的 queue 对象
- 更新 queue 的状态

##### closeQueue

- 从 client 获取最新的 queue 对象
- 更新 queue 的状态

### JobController 控制器

根据 job 获取和构建 pods 信息，然后按照 action 决定创建和销毁 pods.

#### 启动入口 Run

```go
func (cc *jobcontroller) Run(stopCh <-chan struct{}) {
    // 启动 informer
    go cc.jobInformer.Informer().Run(stopCh)
    go cc.podInformer.Informer().Run(stopCh)
    go cc.pvcInformer.Informer().Run(stopCh)
    go cc.pgInformer.Informer().Run(stopCh)
    go cc.svcInformer.Informer().Run(stopCh)
    go cc.cmdInformer.Informer().Run(stopCh)
    go cc.pcInformer.Informer().Run(stopCh)
    go cc.queueInformer.Informer().Run(stopCh)

    // 等待 informer 同步完成
    cache.WaitForCacheSync(stopCh, cc.jobSynced, cc.podSynced, cc.pgSynced,
        cc.svcSynced, cc.cmdSynced, cc.pvcSynced, cc.pcSynced, cc.queueSynced)

    // 监听处理 commands 信号. 
    go wait.Until(cc.handleCommands, 0, stopCh)

    // 并发启动 worker
    var i uint32
    for i = 0; i < cc.workers; i++ {
        go func(num uint32) {
            wait.Until(
                func() {
                    cc.worker(num)
                },
                time.Second,
                stopCh)
        }(i)
    }

    // jobcache 清理
    go cc.cache.Run(stopCh)

    // 处理异常的 task
    go wait.Until(cc.processResyncTask, 0, stopCh)

    klog.Infof("JobController is running ...... ")
}
```

#### worker.processNextReq

volcano 为了提高处理性能，在 jobcontroller 内部抽象了队列数组，启动多个 worker 协程，每个协程绑定一个消费的 queue，而入队时会根据 fnv_hash(namespace/job) 取摸的方式来获取对应的 queue。

通过多队列多 worker 模式来提高处理性能，而且更重要的是可保证 job 级别的有序处理。

```go
func (cc *jobcontroller) worker(i uint32) {
    klog.Infof("worker %d start ...... ", i)

    for cc.processNextReq(i) {
    }
}

func (cc *jobcontroller) processNextReq(count uint32) bool {
    // 获取 worker 对应的 queue 对象.
    queue := cc.queueList[count]
    obj, shutdown := queue.Get()
    if shutdown {
        klog.Errorf("Fail to pop item from queue")
        return false
    }

    req := obj.(apis.Request)
    defer queue.Done(req)

    // 拼装 ns/job-name
    key := jobcache.JobKeyByReq(&req)

    // 如果入队异常，则重新选择入队.
    if !cc.belongsToThisRoutine(key, count) {
        queueLocal := cc.getWorkerQueue(key)
        queueLocal.Add(req)
        return true
    }

    // 从缓存中获取 jobinfo，cache 的数据是由 informer eventhandler 来操作的.
    jobInfo, err := cc.cache.Get(key)
    if err != nil {
        return true
    }

    // 根据 job 状态获取不同的状态处理方法.
    st := state.NewState(jobInfo)
    if st == nil {
        return true
    }

    // 应用 job 中指定的策略
    action := applyPolicies(jobInfo.Job, &req)
    if err := st.Execute(action); err != nil {
        // ...
    }

    queue.Forget(req)

    return true
}
```

#### SyncJob

`pkg/controllers/job/job_controller_actions.go`

初始化 job 相关配置.

- 根据 job 对象从 queue informer lister 里获取 queue 对象.
- initiateJob
  - initJobStatus, init job status
  - createJobIOIfNotExist, 如果 job 没有绑定 pvc，为其分配 pvc
  - createOrUpdatePodGroup, 为 pg 创建 podgroup.
- 遍历 job.spec.tasks 根据当前 jobinfo 状态构建需要创建和删除的 pods 集合.
  - createJobPod 在构建 pod 对象时会赋值 SchedulerName 自定义调度器，默认为 `volcano`
  - 判断 jobInfo 里是否有对应的 pod
    - 如果不存在，则进行创建.
    - 如果存在，则在 pods 字段里删掉, 然后删掉一些缩容后多余的 pods.
- 更新 job 的状态 
- 更新内部的 cache 组件

#### KillJob

`pkg/controllers/job/job_controller_actions.go`

- 遍历 jobinfo 的 pods 集合, 调用 `deleteJobPod` 删除 pod 对象.
- 更新 job 状态
- 删除 job 关联的 podgroup 对象.

#### State

job state 用来处理不同 job 状态下的动作行为.

`NewState` 根据 job 状态构建不同的方法.

```go
// NewState gets the state from the volcano job Phase.
func NewState(jobInfo *apis.JobInfo) State {
    job := jobInfo.Job
    switch job.Status.State.Phase {
    case vcbatch.Pending:
        return &pendingState{job: jobInfo}
    case vcbatch.Running:
        return &runningState{job: jobInfo}
    case vcbatch.Restarting:
        return &restartingState{job: jobInfo}
    case vcbatch.Terminated, vcbatch.Completed, vcbatch.Failed:
        return &finishedState{job: jobInfo}
    case vcbatch.Terminating:
        return &terminatingState{job: jobInfo}
    case vcbatch.Aborting:
        return &abortingState{job: jobInfo}
    case vcbatch.Aborted:
        return &abortedState{job: jobInfo}
    case vcbatch.Completing:
        return &completingState{job: jobInfo}
    }

    return &pendingState{job: jobInfo}
}
```

拿 pending state 待处理方法举例说明，异常状态都走 `killjob`, 而正常状态则走 `syncjob`.

其他 state 实现类似，根据不同的 action 走 killjob 或 syncjob。

```go
type pendingState struct {
    job *apis.JobInfo
}

func (ps *pendingState) Execute(action v1alpha1.Action) error {
    switch action {
    case v1alpha1.RestartJobAction:
        // 如果需要重启, 则在 killjob 里判断 restart 次数，满足阈值则进行 pod 删除.
        return KillJob(ps.job, PodRetainPhaseNone, func(status *vcbatch.JobStatus) bool {
            status.RetryCount++
            status.State.Phase = vcbatch.Restarting
            return true
        })

    case v1alpha1.AbortJobAction:
        // 终止，则直接进行 pod 删除. 
        return KillJob(ps.job, PodRetainPhaseSoft, func(status *vcbatch.JobStatus) bool {
            status.State.Phase = vcbatch.Aborting
            return true
        })
    case v1alpha1.CompleteJobAction:
        // 当任务已完成，则进行收尾删除.
        return KillJob(ps.job, PodRetainPhaseSoft, func(status *vcbatch.JobStatus) bool {
            status.State.Phase = vcbatch.Completing
            return true
        })
    case v1alpha1.TerminateJobAction:
        // 任务终止，也需要进行删除。
        return KillJob(ps.job, PodRetainPhaseSoft, func(status *vcbatch.JobStatus) bool {
            status.State.Phase = vcbatch.Terminating
            return true
        })
    default:
        // 其他 action，都走 syncjob，该逻辑主要是 reconcile 调和.
        return SyncJob(ps.job, func(status *vcbatch.JobStatus) bool {
            if ps.job.Job.Spec.MinAvailable <= status.Running+status.Succeeded+status.Failed {
                status.State.Phase = vcbatch.Running
                return true
            }
            return false
        })
    }
}
```

### PodGroupController 控制器

pg 的逻辑比较简单，就是维护 podgroup pod.

#### 启动入口 Run

启动 pod 和 pg informer，等待着两个 informer 数据同步完毕后，则启动一个协程执行 worker.

```go
func (pg *pgcontroller) Run(stopCh <-chan struct{}) {
    go pg.podInformer.Informer().Run(stopCh)
    go pg.pgInformer.Informer().Run(stopCh)

    cache.WaitForCacheSync(stopCh, pg.podSynced, pg.pgSynced)

    go wait.Until(pg.worker, 0, stopCh)

    klog.Infof("PodgroupController is running ...... ")
}
```

#### processNextReq

```go
func (pg *pgcontroller) worker() {
    for pg.processNextReq() {
    }
}

func (pg *pgcontroller) processNextReq() bool {
    obj, shutdown := pg.queue.Get()
    if shutdown {
        return false
    }

    req := obj.(podRequest)
    defer pg.queue.Done(req)

    // 从 pod informer lister 获取 pg 关联的 pod 对象.
    pod, err := pg.podLister.Pods(req.podNamespace).Get(req.podName)
    if err != nil {
        return true
    }

    // 如果 pg 不存在， 则进行创建
    if err := pg.createNormalPodPGIfNotExist(pod); err != nil {
        pg.queue.AddRateLimited(req)
        return true
    }

    // If no error, forget it.
    pg.queue.Forget(req)

    return true
}
```

## 总结

volcano 社区没以前活跃了，几个月前提交的代码，现在都没有合并。😅