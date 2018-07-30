# Indexing Service 源码之： （1） 任务提交

这里我们将介绍 Overlord 接收客户端任务提交的过程。

# 客户端提交任务

如[文档](http://druid.io/docs/latest/tutorials/tutorial-batch.html)所示，如果我们想要提交一个索引任务，只需要定义好任务的 Json 描述文件，然后通过 HTTP POST 到 Overlord 服务即可：

```bash
curl -X 'POST' -H 'Content-Type:application/json' -d @my-index-task.json OVERLORD_IP:8090/druid/indexer/v1/task
```

# Overlord 接收客户端提交的任务

客户端提交索引任务后， 通过 URL 我们可以从代码中找到 Overlord 接收索引任务的地方是在 `OverlordResource.java` :

```java
@Path("/druid/indexer/v1")
public class OverlordResource
{
...

  @POST
  @Path("/task")
  @Consumes(MediaType.APPLICATION_JSON)
  @Produces(MediaType.APPLICATION_JSON)
  public Response taskPost(
      final Task task,
      @Context final HttpServletRequest req
  )
  {
    ...

    return asLeaderWith(
        taskMaster.getTaskQueue(),
        new Function<TaskQueue, Response>()
        {
          @Override
          public Response apply(TaskQueue taskQueue)
          {
            try {
              taskQueue.add(task);
              return Response.ok(ImmutableMap.of("task", task.getId())).build();
            }
            catch (EntryExistsException e) {
              return Response.status(Response.Status.BAD_REQUEST)
                             .entity(ImmutableMap.of("error", String.format("Task[%s] already exists!", task.getId())))
                             .build();
            }
          }
        }
    );
  }

  private <T> Response asLeaderWith(Optional<T> x, Function<T, Response> f)
  {
    if (x.isPresent()) {
      return f.apply(x.get());
    } else {
      // Encourage client to try again soon, when we'll likely have a redirect set up
      return Response.status(Response.Status.SERVICE_UNAVAILABLE).build();
    }
  }
...
}
```

如上，`taskPost()` 是处理 task POST 请求的方法。

用户提交的 task json 会被反序列化为 `Task`，但是 `Task` 是一个接口类型，它的实现类是哪个呢。

这是因为 Json 反序列化使用了 `JsonSubTypes` 的支持，可以根据 Json 内容来决定反序列化的目标类。如下 `Task` 的定义可知，根据客户端提交的 Json 对象字段 `type` 来决定具体类，如果 `type` 等于 `index_hadoop`，则会将 Json 内容反序列化为 HadoopIndexTask 对象。

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes(value = {
    @JsonSubTypes.Type(name = "append", value = AppendTask.class),
    @JsonSubTypes.Type(name = "merge", value = MergeTask.class),
    @JsonSubTypes.Type(name = "kill", value = KillTask.class),
    @JsonSubTypes.Type(name = "move", value = MoveTask.class),
    @JsonSubTypes.Type(name = "archive", value = ArchiveTask.class),
    @JsonSubTypes.Type(name = "restore", value = RestoreTask.class),
    @JsonSubTypes.Type(name = "index", value = IndexTask.class),
    @JsonSubTypes.Type(name = "index_hadoop", value = HadoopIndexTask.class),
    @JsonSubTypes.Type(name = "hadoop_convert_segment", value = HadoopConverterTask.class),
    @JsonSubTypes.Type(name = "hadoop_convert_segment_sub", value = HadoopConverterTask.ConverterSubTask.class),
    @JsonSubTypes.Type(name = "index_realtime", value = RealtimeIndexTask.class),
    @JsonSubTypes.Type(name = "noop", value = NoopTask.class),
    @JsonSubTypes.Type(name = "version_converter", value = ConvertSegmentBackwardsCompatibleTask.class), // Backwards compat - Deprecated
    @JsonSubTypes.Type(name = "version_converter_sub", value = ConvertSegmentBackwardsCompatibleTask.SubTask.class), // backwards compat - Deprecated
    @JsonSubTypes.Type(name = "convert_segment", value = ConvertSegmentTask.class),
    @JsonSubTypes.Type(name = "convert_segment_sub", value = ConvertSegmentTask.SubTask.class)
})
public interface Task
{
...
}
```

回到 taskPost()  中的 asLeaderWith() 方法：

- 首先通过 taskMaster.getTaskQueue() 获取到 taskQueue ， taskQueue 维护了所有提交的任务;
- 然后 taskQueue.add(task) 把刚刚反序列化得到的 task 加到 taskQueue，然后返回通过 HTTP Response 返回任务提交的结果。

## taskMaster.getTaskQueue()

获取 taskQueue 首先要考虑 leader 选举的问题 ， 由于一个集群中 Overlord 可以有多个，但只有一个会成为 leader 。这里的 `leading` 布尔值表示当前 Overlord 是否获得 leadership （选举的细节请查看 Coordinator 的相关章节）：
- 是则返回 taskQueue，类型是 `TaskQueue`，后面会介绍。
- 否则返回 Optional.absent()，即 `Absent` 对象，从 asLeaderWith() 方法可知，此时将会返回 503 （"Service Unavailable"） HTTP 状态码给客户端。

`TaskMaster.java`：

```java
  public Optional<TaskQueue> getTaskQueue()
  {
    if (leading) {
      return Optional.of(taskQueue);
    } else {
      return Optional.absent();
    }
  }
```

## taskQueue.add(task)

如下， 把 task 加入到 taskQueue 时， 首先会检查当前的执行环境是否合法， 接着：

`code1`：taskStorage 是 TaskStorage 接口，实现是 `MetadataTaskStorage`，taskStorage.insert() 将通过 JDBI 往数据库中插入一条 task 记录。

`code2`：tasks 是 List<Task> ， 用于记录所有新添加的任务。

`code3`：TaskLockbox 用来管理任务锁的 interval 和 version，taskLockbox.add(task) 中只是把 taskId 添加到 activeTasks 记录中，后续发布 segments 时会用到。

`code4`： managementMayBeNecessary 主要用于唤醒等待的线程，比如正在等待新任务到来的管理线程。

`TaskQueue.java`：

```java
  public boolean add(final Task task) throws EntryExistsException
  {
    giant.lock();

    try {
      Preconditions.checkState(active, "Queue is not active!");
      Preconditions.checkNotNull(task, "task");
      Preconditions.checkState(tasks.size() < config.getMaxSize(), "Too many tasks (max = %,d)", config.getMaxSize());

      taskStorage.insert(task, TaskStatus.running(task.getId()));   // code1
      addTaskInternal(task);
      managementMayBeNecessary.signalAll();     // code4
      return true;
    }
    finally {
      giant.unlock();
    }
  }

  private void addTaskInternal(final Task task) {
    tasks.add(task);    // code2
    taskLockbox.add(task);      // code3
  }
```

# TaskQueue 任务管理

前面说到 task 已经被加入到 taskQueue，但是什么时候如何被指派和执行呢？

我们可以看看 TaskQueue ， 它在启动时其实就已经运行了一个线程，不断地循环执行 manage() 方法：

```java
  @LifecycleStart
  public void start()
  {
...
      managerExec.submit(
          new Runnable()
          {
            @Override
            public void run()
            {
              while (true) {
                try {
                  manage();
                  break;
                }
                catch ...
              }
            }
          }
      );
...
  }

```

而 manage() 中主要对前面提到的 `tasks` 进行管理：

1. 运行新提交的未指派任务
2. 清理已经结束的任务

```java

  private void manage() throws InterruptedException
  {
...

    while (active) {
      giant.lock();

      try {
        // Task futures available from the taskRunner
        final Map<String, ListenableFuture<TaskStatus>> runnerTaskFutures = Maps.newHashMap();
        for (final TaskRunnerWorkItem workItem : taskRunner.getKnownTasks()) {
          runnerTaskFutures.put(workItem.getTaskId(), workItem.getResult());
        }
        // Attain futures for all active tasks (assuming they are ready to run).
        // Copy tasks list, as notifyStatus may modify it.
        for (final Task task : ImmutableList.copyOf(tasks)) {
          if (!taskFutures.containsKey(task.getId())) {
            final ListenableFuture<TaskStatus> runnerTaskFuture;
            if (runnerTaskFutures.containsKey(task.getId())) {
              runnerTaskFuture = runnerTaskFutures.get(task.getId());
            } else {
              // Task should be running, so run it.
              final boolean taskIsReady;
              try {
                taskIsReady = task.isReady(taskActionClientFactory.create(task));
              }
              catch (Exception e) {
                log.warn(e, "Exception thrown during isReady for task: %s", task.getId());
                notifyStatus(task, TaskStatus.failure(task.getId()));
                continue;
              }
              if (taskIsReady) {
                log.info("Asking taskRunner to run: %s", task.getId());
                runnerTaskFuture = taskRunner.run(task);
              } else {
                continue;
              }
            }
            taskFutures.put(task.getId(), attachCallbacks(task, runnerTaskFuture));
          }
        }
        // Kill tasks that shouldn't be running
        final Set<String> tasksToKill = Sets.difference(
            runnerTaskFutures.keySet(),
            ImmutableSet.copyOf(
                Lists.transform(
                    tasks,
                    new Function<Task, Object>()
                    {
                      @Override
                      public String apply(Task task)
                      {
                        return task.getId();
                      }
                    }
                )
            )
        );
        if (!tasksToKill.isEmpty()) {
          log.info("Asking taskRunner to clean up %,d tasks.", tasksToKill.size());
          for (final String taskId : tasksToKill) {
            try {
              taskRunner.shutdown(taskId);
            }
            catch (Exception e) {
              log.warn(e, "TaskRunner failed to clean up task: %s", taskId);
            }
          }
        }
        //
        managementMayBeNecessary.awaitNanos(60000000000L /* 60 seconds */);
      }
      finally {
        giant.unlock();
      }
    }
  }
```

如上，只要当前 TaskQueue 仍处于 active， 就不断循环处理 `tasks` 中的每个任务， 如果 runnerTaskFutures 中没有该任务 id 对应的记录，则尝试运行这个任务：

- 通过 taskRunner.run(task) 来指派任务
- 把任务的结果 runnerTaskFuture 加到 taskFutures 管理中

## taskRunner.run(task)

taskRunner 的实现是 TaskRunner 接口类型，根据 `druid.indexer.runner.type` 来绑定实现， 这里的 taskRunner 是 RemoteTaskRunner ，因此我们可以进入其实现方法 RemoteTaskRunner.run() ：

```java
  @Override
  public ListenableFuture<TaskStatus> run(final Task task)
  {
    final RemoteTaskRunnerWorkItem completeTask, runningTask, pendingTask;
    if ((pendingTask = pendingTasks.get(task.getId())) != null) {
      log.info("Assigned a task[%s] that is already pending, not doing anything", task.getId());
      return pendingTask.getResult();
    } else if ((runningTask = runningTasks.get(task.getId())) != null) {
      ZkWorker zkWorker = findWorkerRunningTask(task.getId());
      if (zkWorker == null) {
        log.warn("Told to run task[%s], but no worker has started running it yet.", task.getId());
      } else {
        log.info("Task[%s] already running on %s.", task.getId(), zkWorker.getWorker().getHost());
        TaskAnnouncement announcement = zkWorker.getRunningTasks().get(task.getId());
        if (announcement.getTaskStatus().isComplete()) {
          taskComplete(runningTask, zkWorker, announcement.getTaskStatus());
        }
      }
      return runningTask.getResult();
    } else if ((completeTask = completeTasks.get(task.getId())) != null) {
      return completeTask.getResult();
    } else {
      return addPendingTask(task).getResult();
    }
  }
```

如上，completeTasks、runningTasks、pendingTasks 分别是在内存中维护的不同状态的任务集合，以此判断当前 task 是什么状态：

- 若是 pending 状态，则返回 ListenableFuture<TaskStatus>;
- 若是 running 状态，则判断是否结束并通过 `taskComplete()` 进行任务结束的相关操作，后面会再提到。
- 若是 complete 状态，则返回 ListenableFuture<TaskStatus>;
- 否则说明是新增加的任务，通过 addPendingTask(task) 加入到 pending 队列中，并返回 ListenableFuture<TaskStatus>。

### addPendingTask(task)

```java
  private RemoteTaskRunnerWorkItem addPendingTask(final Task task)
  {
    log.info("Added pending task %s", task.getId());
    final RemoteTaskRunnerWorkItem taskRunnerWorkItem = new RemoteTaskRunnerWorkItem(task.getId(), null, null);
    pendingTaskPayloads.put(task.getId(), task);
    pendingTasks.put(task.getId(), taskRunnerWorkItem);
    runPendingTasks();
    return taskRunnerWorkItem;
  }
```

进入 `runPendingTasks()` ：

```java
  private void runPendingTasks()
  {
    runPendingTasksExec.submit(
        new Callable<Void>()
        {
          @Override
          public Void call() throws Exception
          {
            try {
              List<RemoteTaskRunnerWorkItem> copy = Lists.newArrayList(pendingTasks.values());
              for (RemoteTaskRunnerWorkItem taskRunnerWorkItem : copy) {
                String taskId = taskRunnerWorkItem.getTaskId();
                if (tryAssignTasks.putIfAbsent(taskId, taskId) == null) {
                  try {
                    Task task = pendingTaskPayloads.get(taskId);
                    if (task != null && tryAssignTask(task, taskRunnerWorkItem)) {
                      pendingTaskPayloads.remove(taskId);
                    }
                  }
                  catch (Exception e) {
                    log.makeAlert(e, "Exception while trying to assign task")
                       .addData("taskId", taskRunnerWorkItem.getTaskId())
                       .emit();
                    RemoteTaskRunnerWorkItem workItem = pendingTasks.remove(taskId);
                    if (workItem != null) {
                      taskComplete(workItem, null, TaskStatus.failure(taskId));
                    }
                  }
                  finally {
                    tryAssignTasks.remove(taskId);
                  }
                }
              }
            }
            catch (Exception e) {
              log.makeAlert(e, "Exception in running pending tasks").emit();
            }

            return null;
          }
        }
    );
  }
```

如上， 重点在于 `tryAssignTask(task, taskRunnerWorkItem)` ， 从该方法可以了解到指派任务的细节 ：

```java
  private boolean tryAssignTask(final Task task, final RemoteTaskRunnerWorkItem taskRunnerWorkItem) throws Exception
  {
    Preconditions.checkNotNull(task, "task");
    Preconditions.checkNotNull(taskRunnerWorkItem, "taskRunnerWorkItem");
    Preconditions.checkArgument(task.getId().equals(taskRunnerWorkItem.getTaskId()), "task id != workItem id");

    if (runningTasks.containsKey(task.getId()) || findWorkerRunningTask(task.getId()) != null) {
      log.info("Task[%s] already running.", task.getId());
      return true;
    } else {
      // Nothing running this task, announce it in ZK for a worker to run it
      WorkerBehaviorConfig workerConfig = workerConfigRef.get();
      WorkerSelectStrategy strategy;
      if (workerConfig == null || workerConfig.getSelectStrategy() == null) {
        log.warn("No worker selections strategy set. Using default.");
        strategy = WorkerBehaviorConfig.DEFAULT_STRATEGY;
      } else {
        strategy = workerConfig.getSelectStrategy();
      }

      ZkWorker assignedWorker = null;
      Optional<ImmutableWorkerInfo> immutableZkWorker = null;
      try {
        immutableZkWorker = strategy.findWorkerForTask(     // code1
            config,
            ImmutableMap.copyOf(
                Maps.transformEntries(
                    Maps.filterEntries(
                        zkWorkers, new Predicate<Map.Entry<String, ZkWorker>>()
                        {
                          @Override
                          public boolean apply(Map.Entry<String, ZkWorker> input)
                          {
                            return !lazyWorkers.containsKey(input.getKey()) &&
                                   !workersWithUnacknowledgedTask.containsKey(input.getKey());
                          }
                        }
                    ),
                    new Maps.EntryTransformer<String, ZkWorker, ImmutableWorkerInfo>()
                    {
                      @Override
                      public ImmutableWorkerInfo transformEntry(
                          String key, ZkWorker value
                      )
                      {
                        return value.toImmutable();
                      }
                    }
                )
            ),
            task
        );

        if (immutableZkWorker.isPresent()) {
          if (workersWithUnacknowledgedTask.putIfAbsent(immutableZkWorker.get().getWorker().getHost(), task.getId())
              == null) {
            assignedWorker = zkWorkers.get(immutableZkWorker.get().getWorker().getHost());
            return announceTask(task, assignedWorker, taskRunnerWorkItem);  // code2
          } else {
            log.debug(
                "Lost race to run task [%s] on worker [%s]. Workers to ack tasks are [%s].",
                task.getId(),
                immutableZkWorker.get().getWorker().getHost(),
                workersWithUnacknowledgedTask
            );
          }
        } else {
          log.debug(
              "Unsuccessful task-assign attempt for task [%s] on workers [%s]. Workers to ack tasks are [%s].",
              task.getId(),
              zkWorkers.values(),
              workersWithUnacknowledgedTask
          );
        }

        return false;
      }
      finally {
        if (assignedWorker != null) {
          workersWithUnacknowledgedTask.remove(assignedWorker.getWorker().getHost());   // code3
        }

        if(immutableZkWorker.isPresent()) {
          runPendingTasks();   // code4
        }
      }
    }
  }
```

如上，指派任务主要进行两个操作：

`code1`: 寻找 Worker ， `zkWorkers` 是 Zookeeper 中 `/druid/indexer/announcements` 目录下的所有节点， 过滤之后的 zkWorkers 在通过 strategy.findWorkerForTask() 进行选择， strategy 的默认实现是 `FillCapacityWorkerSelectStrategy` ， 即优先选择当前任务和资源占用低的 Worker 。

`code2`: 如果找到符合条件的 Worker ， 则通过 `announceTask()`  进行该任务的通告 ， 比如往该 Worker 对应的 Zookeeper 目录写入新任务的信息等。

`code3`: 由于 `workersWithUnacknowledgedTask` 记录了正在指派任务的 workers ， 用于处理并发指派任务的情况。 如果 assignedWorker 不等于 null ， 说明该 worker 指派任务成功， 可以从 workersWithUnacknowledgedTask 中删除 assignedWorker。

`code4`: 最后将重新运行前面提到的 runPendingTasks() ， 因为存在并发竞争失败的可能 ， 导致任务没有被指派到 worker ， 任务仍存在于 pendingTasks 记录中 ， 因为 pendingTasks.remove(taskId) 操作是在 announceTask() 中，而竞争失败时是不会执行 announceTask() 的。

#### announceTask()

从上面的代码中，我们已经可以知道 announceTask() 就是进行任务通告的地方了。

这部分的操作主要分为几部分：
- 首先在 Zookeeper 的 `/druid/indexer/tasks/${middleManagerHost}/` 目录下新建节点，节点内容是 Task 的 Json 序列化结果。
- 我当前 task 从 `pendingTasks` 移除， 然后添加到 `runningTasks`。
- 通过 statusLock 等待 Zookeeper 任务节点的更新， 如果超过配置的指派任务超时时间， 则通过 taskComplete() 结束任务。

```java
  private boolean announceTask(
      final Task task,
      final ZkWorker theZkWorker,
      final RemoteTaskRunnerWorkItem taskRunnerWorkItem
  ) throws Exception
  {
    Preconditions.checkArgument(task.getId().equals(taskRunnerWorkItem.getTaskId()), "task id != workItem id");
    final String worker = theZkWorker.getWorker().getHost();
    synchronized (statusLock) {
      if (!zkWorkers.containsKey(worker) || lazyWorkers.containsKey(worker)) {
        // the worker might got killed or has been marked as lazy.
        log.info("Not assigning task to already removed worker[%s]", worker);
        return false;
      }
      log.info("Coordinator asking Worker[%s] to add task[%s]", worker, task.getId());

      CuratorUtils.createIfNotExists(
          cf,
          JOINER.join(indexerZkConfig.getTasksPath(), worker, task.getId()),
          CreateMode.EPHEMERAL,
          jsonMapper.writeValueAsBytes(task),
          config.getMaxZnodeBytes()
      );

      RemoteTaskRunnerWorkItem workItem = pendingTasks.remove(task.getId());
      if (workItem == null) {
        log.makeAlert("WTF?! Got a null work item from pending tasks?! How can this be?!")
           .addData("taskId", task.getId())
           .emit();
        return false;
      }

      RemoteTaskRunnerWorkItem newWorkItem = workItem.withWorker(theZkWorker.getWorker(), null);
      runningTasks.put(task.getId(), newWorkItem);
      log.info("Task %s switched from pending to running (on [%s])", task.getId(), newWorkItem.getWorker().getHost());
      TaskRunnerUtils.notifyStatusChanged(listeners, task.getId(), TaskStatus.running(task.getId()));

      // Syncing state with Zookeeper - don't assign new tasks until the task we just assigned is actually running
      // on a worker - this avoids overflowing a worker with tasks
      Stopwatch timeoutStopwatch = Stopwatch.createStarted();
      while (!isWorkerRunningTask(theZkWorker.getWorker(), task.getId())) {
        final long waitMs = config.getTaskAssignmentTimeout().toStandardDuration().getMillis();
        statusLock.wait(waitMs);
        long elapsed = timeoutStopwatch.elapsed(TimeUnit.MILLISECONDS);
        if (elapsed >= waitMs) {
          log.makeAlert(
              "Task assignment timed out on worker [%s], never ran task [%s]! Timeout: (%s >= %s)!",
              worker,
              task.getId(),
              elapsed,
              config.getTaskAssignmentTimeout()
          );
          taskComplete(taskRunnerWorkItem, theZkWorker, TaskStatus.failure(task.getId()));
          break;
        }
      }
      return true;
    }
  }
```

# Next

至此，我们已经把从客户端提交任务开始到通告任务的主流程源码走了一遍 ， 接下来就是 Worker 获取任务通告和执行。
