## 源码分析 kubernetes cronjob controller 控制器的实现原理

我们可以利用 CronJob 执行基于 crontab 调度的 Job 任务. 创建的 Job 资源是立即执行, 而使用 cronjob 后, 可以周期性的延迟创建 job 任务.

**一个例子**

每隔一分钟访问一下网站.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - curl xiaorui.cc
          restartPolicy: OnFailure
```

**并发性规则**

`.spec.concurrencyPolicy` 也是可选的。它声明了 CronJob 创建的任务执行时发生重叠如何处理。 spec 仅能声明下列规则中的一种：

- Allow: CronJob 允许并发任务执行.
- Forbid: 如果新任务执行时, 老 job 还未完事, 则直接忽略.
- Replace: 如果新任务执行时, 老 job 还未完事, 则清理旧 job, 创建新 job 任务.

### 实例化入口

实例化 cronjob controller 控制器, 传递进去 job 和 cronjob 的 informer 对象, 注册 eventHandler.

在 jobInformer 里注册的 eventHandler 逻辑简单, 从 job 拿到 cronjob 对象然后格式化 key, 再扔到 queue 里, cronjob 也做了同样的逻辑.

```go
func NewControllerV2(jobInformer batchv1informers.JobInformer, cronJobsInformer batchv1informers.CronJobInformer, kubeClient clientset.Interface) (*ControllerV2, error) {
	jm := &ControllerV2{
		queue:       workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "cronjob"),
		jobControl:     realJobControl{KubeClient: kubeClient},
		cronJobControl: &realCJControl{KubeClient: kubeClient},
		jobLister:     jobInformer.Lister(),
		cronJobLister: cronJobsInformer.Lister(),
	}

	// 在 job informer 注册 eventHandler
	jobInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    jm.addJob,
		UpdateFunc: jm.updateJob,
		DeleteFunc: jm.deleteJob,
	})

	// 在 cronjob informer 注册 eventHandler
	cronJobsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			jm.enqueueController(obj)
		},
		UpdateFunc: jm.updateCronJob,
		DeleteFunc: func(obj interface{}) {
			jm.enqueueController(obj)
		},
	})

	return jm, nil
}
```

### 启动入口

启动多个 worker 协程, 每个 worker 先从 queue 获取任务, 然后执行 sync 方法, 如果同步异常则扔到队列里重试. 如果无异常, 则根据下次执行时间把任务放到延迟队列里, 等待下次调度.

```go
func (jm *ControllerV2) Run(ctx context.Context, workers int) {
	// 等待 job 和 cronjob 同步到本地
	if !cache.WaitForNamedCacheSync("cronjob", ctx.Done(), jm.jobListerSynced, jm.cronJobListerSynced) {
		return
	}

	// 启动多个 worker
	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, jm.worker, time.Second)
	}

	// 阻塞, 直到 ctx 被取消
	<-ctx.Done()
}

func (jm *ControllerV2) worker(ctx context.Context) {
	for jm.processNextWorkItem(ctx) {
	}
}

func (jm *ControllerV2) processNextWorkItem(ctx context.Context) bool {
	// 从队列后去 cronjob 的 key
	key, quit := jm.queue.Get()
	if quit {
		return false
	}
	defer jm.queue.Done(key)

	// 执行 cronjob 的核心同步方法
	requeueAfter, err := jm.sync(ctx, key.(string))
	switch {
	case err != nil:
		// 失败, 把任务放到队列里进行重试
		jm.queue.AddRateLimited(key)

	case requeueAfter != nil:
		// 从队列中剔除
		jm.queue.Forget(key)

		// 通过上面的 sync 计算出下次执行的时间, 然后把任务放到 queue 的延迟队列里, 延迟时间为 requeueAfter.
		jm.queue.AddAfter(key, *requeueAfter)
	}
	return true
}
```

### 核心 sync 代码

`sync()` 是 cronjob controller 里核心处理代码的入口, 而 `syncCronJob()` 实现了 cronjob 的主要逻辑.

源码逻辑流程如下:

1. 从 informer lister 获取 cronjob 对象;
2. 获取 cronjob 关联的 jobs 对象集合;
3. 同步 cronjob 的状态, 按照不同的 `ConcurrencyPolicy` 策略, 选择不同的动作, 计算下次执行的时间;
4. 清理已经完成的 jobs;
5. 更新 cronjob 的状态.

```go
func (jm *ControllerV2) sync(ctx context.Context, cronJobKey string) (*time.Duration, error) {
	// 从 key 中拆分 ns 和 name 字段
	ns, name, err := cache.SplitMetaNamespaceKey(cronJobKey)
	if err != nil {
		return nil, err
	}

	// 从 informer lister 获取 cronjob 对象
	cronJob, err := jm.cronJobLister.CronJobs(ns).Get(name)
	switch {
	case errors.IsNotFound(err):
		return nil, nil
	case err != nil:
		// for other transient apiserver error requeue with exponential backoff
		return nil, err
	}

	// 获取 cronjob 关联的 jobs 对象集合
	jobsToBeReconciled, err := jm.getJobsToBeReconciled(cronJob)
	if err != nil {
		return nil, err
	}

	// 获取 cronjob 对象, 下次执行的时间, 是否更新状态等
	cronJobCopy, requeueAfter, updateStatus, err := jm.syncCronJob(ctx, cronJob, jobsToBeReconciled)
	if err != nil {
		if updateStatus {
			// 更新 cronjob 的状态
			if _, err := jm.cronJobControl.UpdateStatus(ctx, cronJobCopy); err != nil {
				return nil, err
			}
		}
		return nil, err
	}

	// 清理已经完成的 jobs.
	if jm.cleanupFinishedJobs(ctx, cronJobCopy, jobsToBeReconciled) {
		updateStatus = true
	}

	// 更新 cronjob 的状态
	if updateStatus {
		if _, err := jm.cronJobControl.UpdateStatus(ctx, cronJobCopy); err != nil {
			return nil, err
		}
	}

	// 如果拿到了下次执行的 duration, 则返回该 duration
	if requeueAfter != nil {
		return requeueAfter, nil
	}
	return nil, nil
}
```

#### syncCronJob 同步定时任务

代码流程如下:

- 遍历 cronjob active 集合, 如果对应的 job 不再存在, 则从 active list 中删除 job 引用. 避免 cronjob 可能永远处于活动模式 ;
- 如果删除时间不为空, 说明该对象已被删除, 后面无需处理了 ;
- 该 cronjob 已暂停则直接退出 ;
- 获取开源 cron 库的调度解释器 ;
- 根据 crontab spec 表达式计算下次执行的时间;
- 如果配置了 ForbidConcurrent 策略, 且当前已经有 job 还在运行, 则直接跳出 ;
- 如果配置了 ReplaceConcurrent 策略, 则需要清理以前还在运行的 Job ;
- 获取 cronjob 对应的 job 模板, 然后创建 job 对象资源 ;
- 在 cronjob 对象里关联 job 对象, 记录 LastScheduleTime 时间和状态等 ;
- 获取下次执行的 duration.
- return

```go
func (jm *ControllerV2) syncCronJob(
	ctx context.Context,
	cronJob *batchv1.CronJob,
	jobs []*batchv1.Job) (*batchv1.CronJob, *time.Duration, bool, error) {

	cronJob = cronJob.DeepCopy()
	now := jm.now()
	updateStatus := false
	timeZoneEnabled := utilfeature.DefaultFeatureGate.Enabled(features.CronJobTimeZone)

	// 创建一个集合映射 job.uid.
	childrenJobs := make(map[types.UID]bool)
	for _, j := range jobs {
		childrenJobs[j.ObjectMeta.UID] = true

		// job 是否在 cronJob active 里
		found := inActiveList(*cronJob, j.ObjectMeta.UID)

		// 如果 job uid 跟 cronjob uid 不相同, 且任务没完成 
		if !found && !IsJobFinished(j) {
			// 获取 cronjob 对象
			cjCopy, err := jm.cronJobControl.GetCronJob(ctx, cronJob.Namespace, cronJob.Name)
			if err != nil {
				return nil, nil, updateStatus, err
			}

			// 再次判断是否相等, 如果相等则使用新拿到的 cronjob 对象
			if inActiveList(*cjCopy, j.ObjectMeta.UID) {
				cronJob = cjCopy
				continue
			}

		} else if found && IsJobFinished(j) {
			// 如果相同, 且 job 已完成
			_, status := getFinishedStatus(j)

			// 从 list 中删除该 job
			deleteFromActiveList(cronJob, j.ObjectMeta.UID)

			// 往 event 里输出该状态
			jm.recorder.Eventf(cronJob, corev1.EventTypeNormal, "SawCompletedJob", "Saw completed job: %s, status: %v", j.Name, status)
			updateStatus = true

		} else if IsJobFinished(j) {
			// 如果该 job 已完成, 则更新时间.
			if cronJob.Status.LastSuccessfulTime == nil {
				cronJob.Status.LastSuccessfulTime = j.Status.CompletionTime
				updateStatus = true
			}
		}
	}

	// 遍历 cronjob active 集合, 如果对应的 job 不再存在, 则从 active list 中删除 job 引用. 避免 cronjob 可能永远处于活动模式.
	for _, j := range cronJob.Status.Active {
		_, found := childrenJobs[j.UID]
		if found {
			continue
		}

		_, err := jm.jobControl.GetJob(j.Namespace, j.Name)
		switch {
		case errors.IsNotFound(err):
			jm.recorder.Eventf(cronJob, corev1.EventTypeNormal, "MissingJob", "Active job went missing: %v", j.Name)

			// 在 active 里有, 但 job 里不存在, 则需要在 cronJob active 里删除, 为了一致性.
			deleteFromActiveList(cronJob, j.UID)
			updateStatus = true
		case err != nil:
			return cronJob, nil, updateStatus, err
		}
	}

	// 如果删除时间不为空, 说明该对象已被删除, 后面无需处理了.
	if cronJob.DeletionTimestamp != nil {
		return cronJob, nil, updateStatus, nil
	}

	// timezone 异常则退出, 输出一条未知时区的错误事件
	if timeZoneEnabled && cronJob.Spec.TimeZone != nil {
		if _, err := time.LoadLocation(*cronJob.Spec.TimeZone); err != nil {
			timeZone := pointer.StringDeref(cronJob.Spec.TimeZone, "")
			jm.recorder.Eventf(cronJob, corev1.EventTypeWarning, "UnknownTimeZone", "invalid timeZone: %q: %s", timeZone, err)
			return cronJob, nil, updateStatus, nil
		}
	}

	// 该 cronjob 已暂停则直接退出.
	if cronJob.Spec.Suspend != nil && *cronJob.Spec.Suspend {
		return cronJob, nil, updateStatus, nil
	}

	// 获取开源 cron 库的调度解释器
	sched, err := cron.ParseStandard(formatSchedule(timeZoneEnabled, cronJob, jm.recorder))
	if err != nil {
		return cronJob, nil, updateStatus, nil
	}

	// 根据 crontab spec 表达式计算下次执行的时间
	scheduledTime, err := getNextScheduleTime(*cronJob, now, sched, jm.recorder)
	if err != nil {
		// 异常则发送 cron spec 解析失败的事件
		jm.recorder.Eventf(cronJob, corev1.EventTypeWarning, "InvalidSchedule", "invalid schedule: %s : %s", cronJob.Spec.Schedule, err)
		return cronJob, nil, updateStatus, nil
	}

	// 如果事件为空, 则尝试使用 nextScheduledTimeDuration 来计算下次时间.
	if scheduledTime == nil {
		t := nextScheduledTimeDuration(*cronJob, sched, now)
		return cronJob, t, updateStatus, nil
	}

	tooLate := false
	// 如果配置 StartingDeadlineSeconds, 则开启 toolate 延后
	if cronJob.Spec.StartingDeadlineSeconds != nil {
		tooLate = scheduledTime.Add(time.Second * time.Duration(*cronJob.Spec.StartingDeadlineSeconds)).Before(now)
	}
	if tooLate {
		t := nextScheduledTimeDuration(*cronJob, sched, now)
		return cronJob, t, updateStatus, nil
	}

	// 如果配置了 ForbidConcurrent 策略, 且当前已经有 job 还在运行, 则直接跳出 
	if cronJob.Spec.ConcurrencyPolicy == batchv1.ForbidConcurrent && len(cronJob.Status.Active) > 0 {
		t := nextScheduledTimeDuration(*cronJob, sched, now)
		return cronJob, t, updateStatus, nil
	}

	// 如果配置了 ReplaceConcurrent 策略, 则需要清理以前还在运行的 Job.
	if cronJob.Spec.ConcurrencyPolicy == batchv1.ReplaceConcurrent {
		for _, j := range cronJob.Status.Active {
			// 获取当前的 job 对象
			job, err := jm.jobControl.GetJob(j.Namespace, j.Name)
			if err != nil {
				return cronJob, nil, updateStatus, err
			}
			// 删除该 job 对象
			if !deleteJob(cronJob, job, jm.jobControl, jm.recorder) {
				return cronJob, nil, updateStatus, fmt.Errorf("could not replace job %s/%s", job.Namespace, job.Name)
			}
			updateStatus = true
		}
	}

	// 获取 cronjob 对应的 job 模板
	jobReq, err := getJobFromTemplate2(cronJob, *scheduledTime)
	if err != nil {
		return cronJob, nil, updateStatus, err
	}

	// 创建 job 任务
	jobResp, err := jm.jobControl.CreateJob(cronJob.Namespace, jobReq)

	// 在 cronjob 对象里关联 job 对象, 记录 LastScheduleTime 时间和状态等
	jobRef, err := getRef(jobResp)
	cronJob.Status.Active = append(cronJob.Status.Active, *jobRef)
	cronJob.Status.LastScheduleTime = &metav1.Time{Time: *scheduledTime}
	updateStatus = true

	// 获取下次调度执行的 duration
	t := nextScheduledTimeDuration(*cronJob, sched, now)
	return cronJob, t, updateStatus, nil
}
```

#### 计算下次调度的时间 nextScheduledTimeDuration

nextScheduledTimeDuration 用来获取下次调度的时间, 代码的逻辑相对有些绕, 核心目的就是为了拿到合理的调度时间.

首先下次的调度不能在 now 之前, 如果在 now 之前则尝试使用下下次的时间. 另外从调度时间点计算 duration 时长时, 加入了 100ms 的抖动, 该抖动应该是为了应对拿到负数 duration.

```go
var (
	nextScheduleDelta = 100 * time.Millisecond
)

func nextScheduledTimeDuration(cj batchv1.CronJob, sched cron.Schedule, now time.Time) *time.Duration {
	// 默认使用创建时间, 通常第一次的时候, 没有上次的调度时间.
	earliestTime := cj.ObjectMeta.CreationTimestamp.Time
	if cj.Status.LastScheduleTime != nil {
		earliestTime = cj.Status.LastScheduleTime.Time
	}

	// 计算最近的时间点
	mostRecentTime, _, err := getMostRecentScheduleTime(earliestTime, now, sched)
	if err != nil {
		mostRecentTime = &now
	} else if mostRecentTime == nil {
		// 为空, 则使用 earliestTime 时间
		mostRecentTime = &earliestTime
	}

	// 依据 mostRecentTime 时间点和 crontab spec 计算出下次 cron 调度时间点, 增加 100ms 的波动, 减去当前时间为 duration 时长
	t := sched.Next(*mostRecentTime).Add(nextScheduleDelta).Sub(now)

	return &t
}

func getMostRecentScheduleTime(earliestTime time.Time, now time.Time, schedule cron.Schedule) (*time.Time, int64, error) {
	t1 := schedule.Next(earliestTime)
	t2 := schedule.Next(t1)

	// 如果 now 在 t1 前面, 直接退出, 可以直接使用 earliestTime 作为时间点计算.
	if now.Before(t1) {
		return nil, 0, nil
	}

	// 如果 now 在 t1 后面, 说明当前的 next 时间不能用了, 需要使用 next next 的时间点, 再来判断.

	// 如果 now 在 next.next 的前面, 可以退出, 使用 t1 的时间.
	if now.Before(t2) {
		return &t1, 1, nil
	}

	// 说实话没看懂啥意思... 😅
	timeBetweenTwoSchedules := int64(t2.Sub(t1).Round(time.Second).Seconds())
	if timeBetweenTwoSchedules < 1 {
		return nil, 0, fmt.Errorf("time difference between two schedules less than 1 second")
	}
	timeElapsed := int64(now.Sub(t1).Seconds())
	numberOfMissedSchedules := (timeElapsed / timeBetweenTwoSchedules) + 1
	t := time.Unix(t1.Unix()+((numberOfMissedSchedules-1)*timeBetweenTwoSchedules), 0).UTC()
	return &t, numberOfMissedSchedules, nil
}
```

**cronjob 如何解决时间临界点的问题 ?**

出现的原因: 

当 time.sub 的时候, 由于时间精度问题导致拿到的 diff 时间会变小, 放到 workqueue delay heap 中等待调度, 当再次被调度时, 有可能在 scheduledTime 之前就被调度起来了. 那么再次求 next 时, 很可能跟上次 scheduledTime 一样.

> 几年前在开发基于 crontab 定时器时, 遇到过该问题. [https://github.com/rfyiamcool/cronlib](https://github.com/rfyiamcool/cronlib)

解决方法:

cronjob 内部会判断当前计算出来的 scheduledTime 是否跟 LastScheduledTime 一致, 如一致则使用下下次的时间. 每次 time.Sub 时会多加 `nextScheduleDelta` 100ms, 这个极大的减少了上面 case 概率的发生. 

#### 清理已完成的 job 对象 cleanupFinishedJobs

清理已经完成状态的 job 对象, removeOldestJobs 最后还是调用 deleteJob 来清理 job.

```go
func (jm *ControllerV2) cleanupFinishedJobs(ctx context.Context, cj *batchv1.CronJob, js []*batchv1.Job) bool {
	if cj.Spec.FailedJobsHistoryLimit == nil && cj.Spec.SuccessfulJobsHistoryLimit == nil {
		return false
	}

	updateStatus := false
	failedJobs := []*batchv1.Job{}
	successfulJobs := []*batchv1.Job{}

	for _, job := range js {
		isFinished, finishedStatus := jm.getFinishedStatus(job)
		if isFinished && finishedStatus == batchv1.JobComplete {
			successfulJobs = append(successfulJobs, job)
		} else if isFinished && finishedStatus == batchv1.JobFailed {
			failedJobs = append(failedJobs, job)
		}
	}

	if cj.Spec.SuccessfulJobsHistoryLimit != nil &&
		jm.removeOldestJobs(cj,
			successfulJobs,
			*cj.Spec.SuccessfulJobsHistoryLimit) {
		updateStatus = true
	}

	if cj.Spec.FailedJobsHistoryLimit != nil &&
		jm.removeOldestJobs(cj,
			failedJobs,
			*cj.Spec.FailedJobsHistoryLimit) {
		updateStatus = true
	}

	return updateStatus
}
```