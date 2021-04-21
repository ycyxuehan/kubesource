# EventHandler

## addAllEventHandlers()

为调度器添加事件处理程序, 对各种通知(`informers`)事件进行测试及调度。

### Pods

#### 已调度pod

```go
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
					return assignedPod(t)
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return assignedPod(pod)
					}
					utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
					return false
				default:
					utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToCache,
				UpdateFunc: sched.updatePodInCache,
				DeleteFunc: sched.deletePodFromCache,
			},
		},
	)
```

这里定义了一个`FilterFunc`用于筛选出已调度的pod。
方法`assignedPod`用于确认pod是否已经分配，其代码如下：

```go
// assignedPod selects pods that are assigned (scheduled and running).
func assignedPod(pod *v1.Pod) bool {
	return len(pod.Spec.NodeName) != 0
}
```

##### sched.addPodToCache

`sched.addPodToCache`被配置为pod的add操作的处理程序。其功能是把pod添加到缓存中。

```go
func (sched *Scheduler) addPodToCache(obj interface{}) {
	pod, ok := obj.(*v1.Pod)
	if !ok {
		klog.ErrorS(nil, "Cannot convert to *v1.Pod", "obj", obj)
		return
	}
	klog.V(3).InfoS("Add event for scheduled pod", "pod", klog.KObj(pod))

	if err := sched.SchedulerCache.AddPod(pod); err != nil {
		klog.ErrorS(err, "Scheduler cache AddPod failed", "pod", klog.KObj(pod))
	}

	sched.SchedulingQueue.AssignedPodAdded(pod)
}
```

`sched.SchedulingQueue.AssignedPodAdded(pod)` 将pod加入调度队列(Added)。这里`sched`只更新缓存，并不进行真正的调度。

***暂时还没看SchedulingQueue***

##### updatePodInCache

`sched.updatePodInCache` 接受两个interface参数，一个是旧pod，一个是新pod

首先判断两个pod是否为一个pod(UID是否一致)，不是则删除旧pod，添加新pod

```go
	if oldPod.UID != newPod.UID {
		sched.deletePodFromCache(oldObj)
		sched.addPodToCache(newObj)
		return
	}
```

如果是同一个pod， 更新pod

```go
	if err := sched.SchedulerCache.UpdatePod(oldPod, newPod); err != nil {
		klog.ErrorS(err, "Scheduler cache UpdatePod failed", "oldPod", klog.KObj(oldPod), "newPod", klog.KObj(newPod))
	}
```

 将pod加入调度队列(Updated)

 ```go
sched.SchedulingQueue.AssignedPodUpdated(newPod)
 ```

##### deletePodFromCache

`sched.deletePodFromCache` 接受一个interface类型参数obj。

首先对obj类型进行判断，并转换为`*v1.Pod`

```go
	var pod *v1.Pod
	switch t := obj.(type) {
	case *v1.Pod:
		pod = t
	case cache.DeletedFinalStateUnknown:
		var ok bool
		pod, ok = t.Obj.(*v1.Pod)
		if !ok {
			klog.ErrorS(nil, "Cannot convert to *v1.Pod", "obj", t.Obj)
			return
		}
	default:
		klog.ErrorS(nil, "Cannot convert to *v1.Pod", "obj", t)
		return
	}
```

从`cache`中删除pod

```go
	if err := sched.SchedulerCache.RemovePod(pod); err != nil {
		klog.ErrorS(err, "Scheduler cache RemovePod failed", "pod", klog.KObj(pod))
	}
```

将删除操作加入队列

```go
sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.AssignedPodDelete)
```

#### 未调度pod

```go
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
					return !assignedPod(t) && responsibleForPod(t, sched.Profiles)
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return !assignedPod(pod) && responsibleForPod(pod, sched.Profiles)
					}
					utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
					return false
				default:
					utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToSchedulingQueue,
				UpdateFunc: sched.updatePodInSchedulingQueue,
				DeleteFunc: sched.deletePodFromSchedulingQueue,
			},
		},
	)
```

这里定义了一个`FilterFunc`用于筛选出未调度的pod。
方法`assignedPod`用于确认pod是否已经分配，而`responsibleForPod`确认pod是否已经被给定调度器调度

```go
func responsibleForPod(pod *v1.Pod, profiles profile.Map) bool {
	return profiles.HandlesSchedulerName(pod.Spec.SchedulerName)
}
```

##### addPodToSchedulingQueue

添加pod到scheduling队列

```go
func (sched *Scheduler) addPodToSchedulingQueue(obj interface{}) {
	pod := obj.(*v1.Pod)
	klog.V(3).InfoS("Add event for unscheduled pod", "pod", klog.KObj(pod))
	if err := sched.SchedulingQueue.Add(pod); err != nil {
		utilruntime.HandleError(fmt.Errorf("unable to queue %T: %v", obj, err))
	}
}
```

##### updatePodInSchedulingQueue

更新scheduling队列中的pod。

跳过一些不需要更新的操作

```go
	if oldPod.ResourceVersion == newPod.ResourceVersion {
		return
	}
	if sched.skipPodUpdate(newPod) {
		return
	}
```

`skipPodUpdate` 检查pod更新是否应该被忽略，以下情况被忽略：

- pod 已经被分配

- pod 仅仅变更了`ResourceVersion`, `Spec.NodeName`, `Annotations`,`ManagedFields`, `Finalizers` 和/或 `Conditions` 字段

##### deletePodFromSchedulingQueue

从scheduling队列中删除pod. 这里多了一个操作：

```go
fwk, err := sched.frameworkForPod(pod)
if err != nil {
	klog.ErrorS(err, "Unable to get profile", "pod", klog.KObj(pod))
	return
}
fwk.RejectWaitingPod(pod.UID)
```

`sched.frameworkForPod` 从`profile`中获取pod对应的调度器。`fwk.RejectWaitingPod(pod.UID)` 拒绝一个waiting状态的pod





