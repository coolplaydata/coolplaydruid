# Indexing Service 源码之： （3） 任务的结束

前面说到，任务结束后， Worker 会更新任务状态到对应的 `taskStatusZNode` ， 从而让 Overlord 能知道某个 Task 的状态变更， 因此 Overlord 必然有一个该目录的监听器。

# 监听任务状态变更

通过搜索代码， 可以在 `RemoteTaskRunner.addWorker()` 找到我们感兴趣的内容，
`RemoteTaskRunner.addWorker()` 这个方法是 RemoteTaskRunner 在 Zookeeper 目录发现到新的 Worker 时调用的， 主要是注册 listener 在这个 Worker 上， 以获得 Worker 上的所有任务状态变更通知 ：

```java
  private ListenableFuture<ZkWorker> addWorker(final Worker worker)
  {
    log.info("Worker[%s] reportin' for duty!", worker.getHost());

    try {
      cancelWorkerCleanup(worker.getHost());

      final String workerStatusPath = JOINER.join(indexerZkConfig.getStatusPath(), worker.getHost());
      final PathChildrenCache statusCache = workerStatusPathChildrenCacheFactory.make(cf, workerStatusPath);
      final SettableFuture<ZkWorker> retVal = SettableFuture.create();
      final ZkWorker zkWorker = new ZkWorker(
          worker,
          statusCache,
          jsonMapper
      );

      // Add status listener to the watcher for status changes
      zkWorker.addListener(
          new PathChildrenCacheListener()
          {
            @Override
            public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception
            {
              final String taskId;
              final RemoteTaskRunnerWorkItem taskRunnerWorkItem;
              synchronized (statusLock) {
                try {
                  switch (event.getType()) {
                    case CHILD_ADDED:
                    case CHILD_UPDATED:
                      taskId = ZKPaths.getNodeFromPath(event.getData().getPath());
                      final TaskAnnouncement announcement = jsonMapper.readValue(
                          event.getData().getData(), TaskAnnouncement.class
                      );

                      log.info(
                          "Worker[%s] wrote %s status for task [%s] on [%s]",
                          zkWorker.getWorker().getHost(),
                          announcement.getTaskStatus().getStatusCode(),
                          taskId,
                          announcement.getTaskLocation()
                      );

                      // Synchronizing state with ZK
                      statusLock.notifyAll();

                      final RemoteTaskRunnerWorkItem tmp;
                      if ((tmp = runningTasks.get(taskId)) != null) {
                        taskRunnerWorkItem = tmp;
                      } else {
                        final RemoteTaskRunnerWorkItem newTaskRunnerWorkItem = new RemoteTaskRunnerWorkItem(
                            taskId,
                            zkWorker.getWorker(),
                            TaskLocation.unknown()
                        );
                        final RemoteTaskRunnerWorkItem existingItem = runningTasks.putIfAbsent(
                            taskId,
                            newTaskRunnerWorkItem
                        );
                        if (existingItem == null) {
                          log.warn(
                              "Worker[%s] announced a status for a task I didn't know about, adding to runningTasks: %s",
                              zkWorker.getWorker().getHost(),
                              taskId
                          );
                          taskRunnerWorkItem = newTaskRunnerWorkItem;
                        } else {
                          taskRunnerWorkItem = existingItem;
                        }
                      }

                      if (!announcement.getTaskLocation().equals(taskRunnerWorkItem.getLocation())) {
                        taskRunnerWorkItem.setLocation(announcement.getTaskLocation());
                        TaskRunnerUtils.notifyLocationChanged(listeners, taskId, announcement.getTaskLocation());
                      }

                      if (announcement.getTaskStatus().isComplete()) {
                        taskComplete(taskRunnerWorkItem, zkWorker, announcement.getTaskStatus());
                        runPendingTasks();
                      }
                      break;
                    case CHILD_REMOVED:
                      taskId = ZKPaths.getNodeFromPath(event.getData().getPath());
                      taskRunnerWorkItem = runningTasks.remove(taskId);
                      if (taskRunnerWorkItem != null) {
                        log.info("Task[%s] just disappeared!", taskId);
                        taskRunnerWorkItem.setResult(TaskStatus.failure(taskId));
                        TaskRunnerUtils.notifyStatusChanged(listeners, taskId, TaskStatus.failure(taskId));
                      } else {
                        log.info("Task[%s] went bye bye.", taskId);
                      }
                      break;
                    case INITIALIZED:
                      if (zkWorkers.putIfAbsent(worker.getHost(), zkWorker) == null) {
                        retVal.set(zkWorker);
                      } else {
                        final String message = StringUtils.format(
                            "WTF?! Tried to add already-existing worker[%s]",
                            worker.getHost()
                        );
                        log.makeAlert(message)
                           .addData("workerHost", worker.getHost())
                           .addData("workerIp", worker.getIp())
                           .emit();
                        retVal.setException(new IllegalStateException(message));
                      }
                      runPendingTasks();
                      break;
                    case CONNECTION_SUSPENDED:
                    case CONNECTION_RECONNECTED:
                    case CONNECTION_LOST:
                      // do nothing
                  }
                }
                catch (Exception e) {
                  log.makeAlert(e, "Failed to handle new worker status")
                     .addData("worker", zkWorker.getWorker().getHost())
                     .addData("znode", event.getData().getPath())
                     .emit();
                }
              }
            }
          }
      );
      zkWorker.start();
      return retVal;
    }
    catch (Exception e) {
      throw Throwables.propagate(e);
    }
  }
```

在细节上就是通过 `zkWorker.addListener()` 在 `workerStatusPath` 上增加 listener ， 其中 `workerStatusPath` 是 `/druid/indexer/status/${middleManagerHost}/`， 这个目录下面的 ZNode 就是我们提到的 `taskStatusZNode`。
而 `CHILD_UPDATED` 事件对应的是 `taskStatusZNode` 的更新事件， 因此我们可以关注一下这部分的处理逻辑。

- 首先肯定是 `taskStatusZNode` 的数据反序列化为 `TaskAnnouncement`，  其中 taskStatusZNode 的节点名就是 taskId
- 然后通过 announcement.getTaskStatus() 判断该任务是否已经完成， 是则调用 `RemoteTaskRunner.taskComplete()` 进行该任务完成后的处理。

`RemoteTaskRunner.taskComplete()` 主要完成以下工作：

- 在 `RemoteTaskRunner.completeTasks` 这个 `Map` 中增加这个 Task 记录
- 在 `RemoteTaskRunner.runningTasks` 中移除这个 Task 记录
- 记录这个 worker 的任务失败次数， 达到阈值时加入到 `blackListedWorkers` 黑名单。

`RemoteTaskRunner.taskComplete()` ：

```java
  private void taskComplete(
      RemoteTaskRunnerWorkItem taskRunnerWorkItem,
      ZkWorker zkWorker,
      TaskStatus taskStatus
  )
  {
    Preconditions.checkNotNull(taskRunnerWorkItem, "taskRunnerWorkItem");
    Preconditions.checkNotNull(taskStatus, "taskStatus");
    if (zkWorker != null) {
      log.info(
          "Worker[%s] completed task[%s] with status[%s]",
          zkWorker.getWorker().getHost(),
          taskStatus.getId(),
          taskStatus.getStatusCode()
      );
      // Worker is done with this task
      zkWorker.setLastCompletedTaskTime(DateTimes.nowUtc());
    } else {
      log.info("Workerless task[%s] completed with status[%s]", taskStatus.getId(), taskStatus.getStatusCode());
    }

    // Move from running -> complete
    completeTasks.put(taskStatus.getId(), taskRunnerWorkItem);
    runningTasks.remove(taskStatus.getId());

    // Update success/failure counters
    if (zkWorker != null) {
      if (taskStatus.isSuccess()) {
        zkWorker.resetContinuouslyFailedTasksCount();
        if (blackListedWorkers.remove(zkWorker)) {
          zkWorker.setBlacklistedUntil(null);
          log.info("[%s] removed from blacklist because a task finished with SUCCESS", zkWorker.getWorker());
        }
      } else if (taskStatus.isFailure()) {
        zkWorker.incrementContinuouslyFailedTasksCount();
      }

      // Blacklist node if there are too many failures.
      synchronized (blackListedWorkers) {
        if (zkWorker.getContinuouslyFailedTasksCount() > config.getMaxRetriesBeforeBlacklist() &&
            blackListedWorkers.size() <= zkWorkers.size() * (config.getMaxPercentageBlacklistWorkers() / 100.0) - 1) {
          zkWorker.setBlacklistedUntil(DateTimes.nowUtc().plus(config.getWorkerBlackListBackoffTime()));
          if (blackListedWorkers.add(zkWorker)) {
            log.info(
                "Blacklisting [%s] until [%s] after [%,d] failed tasks in a row.",
                zkWorker.getWorker(),
                zkWorker.getBlacklistedUntil(),
                zkWorker.getContinuouslyFailedTasksCount()
            );
          }
        }
      }
    }

    // Notify interested parties
    taskRunnerWorkItem.setResult(taskStatus);
    TaskRunnerUtils.notifyStatusChanged(listeners, taskStatus.getId(), taskStatus);
  }
```

listener 的处理到这里就结束了， 但是你一定会有疑问， 如果一直有任务在跑， 那么 RemoteTaskRunner.completeTasks 中的记录肯定会越积越多直到内存泄漏。 因此， Overlord 必然会在某个地方进行了这方面的清理工作。

# ListenableFuture<TaskStatus> 和 FutureCallback

带着这个问题， 我们现在回想一下前面的“任务提交”部分， `TaskQueue.taskFutures` 类型是 `Map<String, ListenableFuture<TaskStatus>>` ， 它记录了每个 `RemoteTaskRunner.run(Task)` 返回的 `ListenableFuture<TaskStatus>`，  而这个 `ListenableFuture<TaskStatus>` 在 TaskQueue.attachCallbacks() 中注册了一个 FutureCallback ， 当异步执行结束时， 将调用 `notifyStatus()` 方法：

- 调用 taskRunner.shutdown(task.getId()) 关闭任务
- `TaskQueue.tasks` 记录中移除这个 TaskId
- 调用 `TaskStorage.setStatus()` 方法更新这个任务的状态等元信息到数据库

## taskRunner.shutdown(taskId)

这个方法用来关闭 taskId 对应的任务：

```java
  public void shutdown(final String taskId)
  {
    if (!lifecycleLock.awaitStarted(1, TimeUnit.SECONDS)) {
      log.info("This TaskRunner is stopped or not yet started. Ignoring shutdown command for task: %s", taskId);
    } else if (pendingTasks.remove(taskId) != null) {
      pendingTaskPayloads.remove(taskId);
      log.info("Removed task from pending queue: %s", taskId);
    } else if (completeTasks.containsKey(taskId)) {
      cleanup(taskId);
    } else {
      final ZkWorker zkWorker = findWorkerRunningTask(taskId);

      if (zkWorker == null) {
        log.info("Can't shutdown! No worker running task %s", taskId);
        return;
      }
      URL url = null;
      try {
        url = makeWorkerURL(zkWorker.getWorker(), StringUtils.format("/task/%s/shutdown", taskId));
        final StatusResponseHolder response = httpClient.go(
            new Request(HttpMethod.POST, url),
            RESPONSE_HANDLER,
            shutdownTimeout
        ).get();

        log.info(
            "Sent shutdown message to worker: %s, status %s, response: %s",
            zkWorker.getWorker().getHost(),
            response.getStatus(),
            response.getContent()
        );

        if (!HttpResponseStatus.OK.equals(response.getStatus())) {
          log.error("Shutdown failed for %s! Are you sure the task was running?", taskId);
        }
      }
      catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new RE(e, "Interrupted posting shutdown to [%s] for task [%s]", url, taskId);
      }
      catch (Exception e) {
        throw new RE(e, "Error in handling post to [%s] for task [%s]", zkWorker.getWorker().getHost(), taskId);
      }
    }
  }
```

进入其中的 cleanup(taskId) ， 从方法名可知，这里就是具体清理任务的地方了 ：

- 首先从 completeTasks 内存记录中删除这个 task （终于解答了心头的疑问）
- 再去 Zookeeper 中删除 `taskStatusZNode`

```java
  private void cleanup(final String taskId)
  {
    if (!lifecycleLock.awaitStarted(1, TimeUnit.SECONDS)) {
      return;
    }
    final RemoteTaskRunnerWorkItem removed = completeTasks.remove(taskId);
    final Worker worker = removed.getWorker();
    if (removed == null || worker == null) {
      log.makeAlert("WTF?! Asked to cleanup nonexistent task")
         .addData("taskId", taskId)
         .emit();
    } else {
      final String workerId = worker.getHost();
      log.info("Cleaning up task[%s] on worker[%s]", taskId, workerId);
      final String statusPath = JOINER.join(indexerZkConfig.getStatusPath(), workerId, taskId);
      try {
        cf.delete().guaranteed().forPath(statusPath);
      }
      catch (KeeperException.NoNodeException e) {
        log.info("Tried to delete status path[%s] that didn't exist! Must've gone away already?", statusPath);
      }
      catch (Exception e) {
        throw Throwables.propagate(e);
      }
    }
  }
```

# TaskQueue.manage()

除了 FutureCallback， 在“任务提交”时我们提到过 TaskQueue 有一个线程， 在死循环里不断执行 `manage()` 方法（源码参看“任务提交”那部分的页面）， 该方法会通过比较来得到一个 `tasksToKill` （类型是 `Set<String>`）， 记录要停止和清理的任务， 以保持多个内存记录中的数据一致。

`tasksToKill` 是这样来的：

- `runnerTaskFutures` = `pendingTasks` + `runningTasks` + `completeTasks`
- `tasksToKill` = `tasks` 和 `runnerTaskFutures` 的 taskId 相互补集


那么接下来就是对 tasksToKill 的每个 taskId， 调用上面的 `RemoteTaskRunner.shutdown()`。
