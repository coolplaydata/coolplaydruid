# Indexing Service 源码之： （2） Middle Manager 任务执行

前面提到， Overlord 下发任务时会通过 `announceTask()` 将任务写入 Zookeeper 相应目录。现在 Middle Manager 就可利用 Zookeeper 的监听机制， 接收到新指派的任务信息， 开始任务的执行。

# 监听 Zookeeper 中的任务发现目录

`Overlord` 是把新任务写到了 Zookeeper 的 `/druid/indexer/tasks/${middleManagerHost}/` 目录下，因此 `Middle Manager` 必然会监听该目录。监听的注册是在 `WorkerTaskMonitor.start()` 中， 即 `Middle Manager` 在启动的时候就会调用 `registerRunListener()` 进行 Zookeeper 的监听器注册：

```java
  public void start() throws Exception
  {
    synchronized (lifecycleLock) {
      Preconditions.checkState(!started, "already started");
      Preconditions.checkState(!exec.isShutdown(), "already stopped");
      started = true;

      try {
        restoreRestorableTasks();
        cleanupStaleAnnouncements();
        registerRunListener();
        registerLocationListener();
        pathChildrenCache.start();
        exec.submit(
            new Runnable()
            {
              @Override
              public void run()
              {
                mainLoop();
              }
            }
        );

        log.info("Started WorkerTaskMonitor.");
        started = true;
      }
      catch (InterruptedException e) {
        throw e;
      }
      catch (Exception e) {
        log.makeAlert(e, "Exception starting WorkerTaskMonitor")
           .emit();
        throw e;
      }
    }
  }
```

其中 `registerRunListener()` 方法主要定义了 `CHILD_ADDED` 的事件处理 ：

```java
  private void registerRunListener()
  {
    pathChildrenCache.getListenable().addListener(
        new PathChildrenCacheListener()
        {
          @Override
          public void childEvent(CuratorFramework curatorFramework, PathChildrenCacheEvent pathChildrenCacheEvent)
              throws Exception
          {
            if (pathChildrenCacheEvent.getType().equals(PathChildrenCacheEvent.Type.CHILD_ADDED)) {
              final Task task = jsonMapper.readValue(
                  cf.getData().forPath(pathChildrenCacheEvent.getData().getPath()),
                  Task.class
              );

              notices.add(new RunNotice(task));
            }
          }
        }
    );
  }
```

如上，当任务发现目录下触发 `CHILD_ADDED` 事件时， 即有新指派的 Task , 这里定义的 Listener 将把这个节点的数据反序列化为 Task， 封装入 WorkerTaskMonitor.RunNotice 后追加到 notices 队列。

# notices 队列

notices 是个 `BlockingQueue` 队列， 把新任务追加到这个队列后，那自然需要有线程消费它。

如前面的 start() 所示， Middle Manager 在启动时会提交了一个 Runnable 用来执行 mainLoop() 方法， 该方法中不断的循环从 notices 队列中获取 Notice， 然后调用其 handle() 方法处理 ：

```java
  private void mainLoop()
  {
    try {
      while (!Thread.currentThread().isInterrupted()) {
        final Notice notice = notices.take();

        try {
          notice.handle();
        }
        catch (InterruptedException e) {
          // Will be caught and logged in the outer try block
          throw e;
        }
        catch (Exception e) {
          log.makeAlert(e, "Failed to handle notice")
             .addData("noticeClass", notice.getClass().getSimpleName())
             .addData("noticeTaskId", notice.getTaskId())
             .emit();
        }
      }
    }
    catch (InterruptedException e) {
      log.info("WorkerTaskMonitor interrupted, exiting.");
    }
    finally {
      doneStopping.countDown();
    }
  }
```

# RunNotice

从前面可以知道， Task 被封装进了 `RunNotice，` `RunNotice` 又继承了 `Notice`， 因此 mainLoop() 方法中将从队列里获取 `RunNotice`， 然后调用其 `handle()` 方法来执行任务。

`handle()` 的主要内容是：

- 判断如果该 task 处于运行中，则删除 task 在 Zookeeper 的任务发现目录下的对应节点，然后返回。

否则：

- 更新 Zookeeper 中的任务状态节点内容为 `运行中`， 路径是 `/druid/indexer/status/${middleManagerHost}/${task_status_node}`， 我们暂且称之为 `taskStatusZNode` 。
- 删除 task 在任务发现目录下的对应节点。
- `taskRunner.run(task)` 将启动一个 JVM 来运行接收到的任务。
- `addRunningTask()` 将记录 task 到 `running` （Map<String, TaskDetails>） 中，并定义任务异步执行结果的回调处理， 主要是将 task 和 result 封装为 `StatusNotice`， 加入到 notices 队列中。

```java
  private class RunNotice implements Notice
  {
...

    @Override
    public void handle() throws Exception
    {
      if (running.containsKey(task.getId())) {
        log.warn(
            "Got run notice for task [%s] that I am already running...",
            task.getId()
        );
        workerCuratorCoordinator.removeTaskRunZnode(task.getId());
        return;
      }

      log.info("Submitting runnable for task[%s]", task.getId());

      workerCuratorCoordinator.updateTaskStatusAnnouncement(
          TaskAnnouncement.create(
              task,
              TaskStatus.running(task.getId()),
              TaskLocation.unknown()
          )
      );

      log.info("Affirmative. Running task [%s]", task.getId());
      workerCuratorCoordinator.removeTaskRunZnode(task.getId());
      final ListenableFuture<TaskStatus> future = taskRunner.run(task);
      addRunningTask(task, future);
    }
  }
```

`addRunningTask(task, future)` ：

```java
  private void addRunningTask(final Task task, final ListenableFuture<TaskStatus> future)
  {
    running.put(task.getId(), new TaskDetails(task));
    Futures.addCallback(
        future,
        new FutureCallback<TaskStatus>()
        {
          @Override
          public void onSuccess(TaskStatus result)
          {
            notices.add(new StatusNotice(task, result));
          }

          @Override
          public void onFailure(Throwable t)
          {
            notices.add(new StatusNotice(task, TaskStatus.failure(task.getId())));
          }
        }
    );
  }
```

## taskRunner.run(task)

前面 `taskRunner.run(task)` 中的 taskRunner 实现是 `ForkingTaskRunner`， 顾名思义， 将会新起一个进程来运行， 其 run() 方法代码太长不方便贴出来， 主要内容是定义并提交运行一个 Callable ， 这个 Callable 返回 TaskStatus，`call()` 的核心内容是：

- 为当前要运行的任务建立本地临时目录，并在该目录中创建 task.json 和 status.json 文件（下一步会用到）
- 使用 `ProcessBuilder` 构建一个 java 命令， 该命令的内容大概是 `java -cp ...  io.druid.cli.Main internal peon .../task.json .../status.json`， 这个进程就是架构图中的 `Peon` 了， 最终的执行是调用 `Task.run()` 的具体实现（比如 HadoopIndexTask、KafkaIndexTask 等实现）， Task 信息可从 `task.json` 文件读取。
- 执行上一步构建的 java 命令启动一个执行任务的 JVM， 通过 `java.lang.UNIXProcess.waitFor()` 等待该 JVM 执行结束，获取它的进程退出状态码。
- 最后是清理工作， 比如删除任务临时目录等等。

## StatusNotice 处理任务执行结果

StatusNotice 是和状态变化有关的通知， 处理逻辑所在的方法 handle() ：

```java
public void handle() throws Exception
    {
      final TaskDetails details = running.get(task.getId());

...
      if (!status.isComplete()) {
        log.warn(
            "WTF?! Got status notice for task [%s] that isn't complete (status = [%s])...",
            task.getId(),
            status.getStatusCode()
        );
        return;
      }

      details.status = status.withDuration(System.currentTimeMillis() - details.startTime);

      try {
        workerCuratorCoordinator.updateTaskStatusAnnouncement(
            TaskAnnouncement.create(
                details.task,
                details.status,
                details.location
            )
        );
        log.info(
            "Job's finished. Completed [%s] with status [%s]",
            task.getId(),
            status.getStatusCode()
        );
      }
      catch (InterruptedException e) {
        throw e;
      }
      catch (Exception e) {
        log.makeAlert(e, "Failed to update task announcement")
           .addData("task", task.getId())
           .emit();
      }
      finally {
        running.remove(task.getId());
      }
    }
```

如上， 当任务执行完成时， 首先更新 Zookeeper 中的对应任务情况， 比如运行时间、当前运行状态、任务位置（host、post）等等，然后从 running 中（Map<String, TaskDetails>）删除该任务 Id 对应的记录 。


# Next

综上可知，Middle Manager 的一个任务执行完成之后，将同步更新 Zookeeper 中该任务的状态，但这个任务还需要 Overlord 做最后的处理，具体请看下一部分的内容。
