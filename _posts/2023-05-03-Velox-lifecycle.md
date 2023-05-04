---
layout: single
title: Velox lifecycle
date: 2023-05-04 00:00:00 +0800
categories: 源码
tags: Velox Concurency
---

目前项目里采用了类似Velox的运行框架，其中出现了特别多问题。这里会总结Velox中最重要的三个类`Task`，`Driver`，`Operator`的生命周期。

### Task和Driver循环依赖

下面画了下`Task`，`Driver`，`Operator`三个之间的关系，结合下面源码中的介绍`velox/exec/TaskDriverOperatorLifecycle.md`食用更佳。(这个文档中有部分说明已经和最新代码不匹配了)

```
Task                              Driver
┌──────────────────────────┐ 1  ┌────────────────────────────┐             Operator
│vector<shared_ptr<Driver>>├────►unique_ptr<DriverCtx>       │  4  ┌───────────────────────┐
└────────────▲─────────────┘    │vector<unique_ptr<Operator>>├─────►unique_ptr<OperatorCtx>│
             │                  └──────────────┬─────────────┘     └───────────┬───────────┘
             │                                 │                               │
            3│                                2│                              5│
             │                                 │                               │
             │                                 ▼                               ▼
             │                             DriverCtx ──────────┐          OperatorCtx
             │                        ┌────────────────┐       │         ┌────────────┐
             │                        │DriverId        │       └──────── │DriverCtx*  │
             │                        │PipelineId      │                 │PlanNodeId  │
             │                        │SplitGroupId    │                 │OperatorId  │
             │                        │PartitionId     │                 │OperatorType│
             └────────────────────────┤shared_ptr<Task>│                 └────────────┘
                                      │Driver*         │
                                      └────────────────┘

```

图中12345的引用关系都是在创建时候完成，但123形成了个循环引用，注意`Task`和`Driver`互相持有了对方的`shared_ptr`。

### Driver的退出

要理解这个循环引用是如何解除的，必须先理解`Driver`是有哪几种方式退出的，这里总结了下核心的两种退出路径

1. `Driver::close -> Task::removeDriver`: 由`Driver`负责主动关闭，这种情况大多发生在其他`Driver`在执行过程中出现错误或者异常，修改`Task`的状态为`TaskState::kFailed`(又或者是被用户主动取消，将状态修改为`TaskState::kCanceled`)。当前的`Driver`有以下几种情况:
    - `Driver`正在执行，在这次`Driver::runInternal`执行完成之后，会发现`Task`状态有异常，主动关闭自身
    - `Driver`之前被block了，重新入队列开始执行之后，`Driver::runInternal`刚开始也会检查Task的状态，发现有异常之后也会主动关闭自身
2. `Task::terminate -> Driver::closeByTask`: 由Task负责关闭，调用`Task::terminate`的情况有很多:
    - `Driver`正常执行之后，调用`Driver::close`，然后调用`Task::removeDriver`。当所有`Driver`都执行完成后，会由最后完成的`Driver`调用`Task::terminate(TaskState::kFinished)`
    - 某个`Driver`在运行过程中发生错误或者异常，会调用`Task::setError`，然后调用`Task::terminate(TaskState::kFailed)`
    - 用户主动取消`Task`，此时会调用`Task::terminate(TaskState::kCanceled)`

    但无论上面那种情况，在`Task::terminate`函数中都只会由`Task`关掉off-thread的`Driver`，而其他on-thread或者是blocked的`Driver`会由`Driver`自身进行关闭，这是目前velox退出机制中最重要的两点之一。

    > 另外一点就是Task的生命周期一定要比Driver和Operator长
    >

这两条路径合在一起，就是`Driver`的退出路径。每个`Driver`在退出之后，这个`Driver`在`Task`中`vector<shared_ptr<Driver>> drivers_`的对应指针会被置为`nullptr`，也就把文章一开始画的依赖关系图中`1` (也就是`Task -> Driver`)这条链路掐断了。

### 路径一

接下来我们看下第一个退出路径:

```cpp
void Driver::close() {
  if (closed_) {
    // Already closed.
    return;
  }
  if (!isOnThread() && !isTerminated()) {
    LOG(FATAL) << "Driver::close is only allowed from the Driver's thread";
  }
  addStatsToTask();
  for (auto& op : operators_) {
    op->close();
  }
  closed_ = true;
  Task::removeDriver(ctx_->task, this);
}

```

在`Driver::close`中会调用`Operator::close`把自身所持有的Operator都关闭掉:

```cpp

virtual void Operator::close() {
  input_ = nullptr;
  results_.clear();
  if (operatorCtx_->pool()->getMemoryUsageTracker() != nullptr) {
    // Release the unused memory reservation on close.
    operatorCtx_->pool()->getMemoryUsageTracker()->release();
  }
}

```

然后就会调用`Task::removeDriver`，在`Task`中把当前`Driver`自身置为`nullptr`，代表外界不再能通过`Task`访问到该`Driver`.

```cpp
void Task::removeDriver(std::shared_ptr<Task> self, Driver* driver) {
  bool foundDriver = false;
  bool allFinished = true;
  {
    std::lock_guard<std::mutex> taskLock(self->mutex_);
    for (auto& driverPtr : self->drivers_) {
      if (driverPtr.get() != driver) {
        continue;
      }

      // ...

      // Release the driver, note that after this 'driver' is invalid.
      driverPtr = nullptr;
      self->driverClosedLocked();

      allFinished = self->checkIfFinishedLocked();

      // ...

      foundDriver = true;
      break;
    }

    // ...
  }

  if (!foundDriver) {
    LOG(WARNING) << "Trying to remove a Driver twice from its Task";
  }

  if (allFinished) {
    self->terminate(TaskState::kFinished);
  }
}

```

注意此时`Driver`还通过`DriverCtx`持有`Task`的`shared_ptr`。如果所有`Driver`都调用过`removeDriver`后，`allFinished`就会为`true`，就会调用`Task::terminate`。

### 路径二

不管是上面提到的正常退出的路径，还是`Driver`执行过程中出错或被取消，最终都会调用`Task::terminate`进行`Task`的清理。在`terminate`中有一个非常重要的机制，它只会停掉所有off-thread的`Driver`，随后就唤醒外面等待`Task`的future了，告知其这个`Task`已经执行完成(状态可能是成功或者失败)。其他on-thread的`Driver`有几种情况：

- 正在执行，当出`Driver::runInternal`作用域时，会由`CancelGuard`负责完成清理。具体原理就是调用`Task::leave`检查这个`Driver`归属的`Task`是否还应该继续执行，如果返回`StopReason::kTerminate`，就表示这个`Driver`可以退出了。
- 一个`Driver`入队列时，在`Driver::runInternal`一开始通过调用`Task::enter`检查这个`Driver`归属的`Task`状态，如果返回`StopReason::kTerminate`，就表示这个`Driver`可以退出了。
- 还有一种特殊情况就是之前被block的`Driver`，这些`Driver`被保存在`BlockingState`里面，当其中的`Operator`不再被block时，对应的future就会被唤醒，然后重新入队列。入队列之后同样也会检查这个`Driver`归属的`Task`状态，原理和入队列的情况是一样的。

    ```cpp
    void BlockingState::setResume(std::shared_ptr<BlockingState> state) {
      VELOX_CHECK(!state->driver_->isOnThread());
      auto& exec = folly::QueuedImmediateExecutor::instance();
      std::move(state->future_)
          .via(&exec)
          .thenValue([state](auto&& /* unused */) {
           // ...
            Driver::enqueue(state->driver_);
          })
          .thenError(folly::tag_t<std::exception>{}, [state](std::exception const& e) {
                state->driver_->setError(std::make_exception_ptr(e));
          });
    }

    ```

    > 另外`BlockingState`里面还持有一个`Driver`的`shared_ptr`，有可能这个`shared_ptr`是对应`Driver`的最后一个引用，在`BlockingState`析构时候，会顺带析构`Driver`。
    >

然后我们具体看下`terminate`的实现，其主要工作就是遍历`Task`持有的所有`Driver`，找到其中所有off-thread的`Driver`，并把其置为nullptr，在这个过程中还会收集其他需要由`Task`负责清理的资源，比如split，split group，exchange，bridge等等。收集到这些需要清理的资源后，会对每个off-thread的`Driver`调用`closeByTask`，并通知外界这个`Task`的状态已经发生了改变，然后依次清理其他资源，最后返回一个future，这个future在所有`Driver`都退出后会被唤醒。其中`enterForTerminateLocked`这个函数非常关键，它控制了哪些`Driver`是由`Task`负责close，哪些则是由`Driver`自身来close:

```cpp
ContinueFuture Task::terminate(TaskState terminalState) {
  std::vector<std::shared_ptr<Driver>> offThreadDrivers;
  TaskCompletionNotifier completionNotifier;
  std::vector<std::shared_ptr<ExchangeClient>> exchangeClients;
  {
    std::lock_guard<std::mutex> l(mutex_);
    if (taskStats_.executionEndTimeMs == 0) {
      taskStats_.executionEndTimeMs = getCurrentTimeMs();
    }
    // 如果Task的状态已经被设置为其他状态 会返回一个future 当所有Driver都已经退出之后 会唤醒这个future
    if (not isRunningLocked()) {
      return makeFinishFutureLocked("Task::terminate");
    }
    state_ = terminalState;

    // ...

    // Drivers that are on thread will see this at latest when they go off
    // thread.
    terminateRequested_ = true;
    // The drivers that are on thread will go off thread in time and
    // 'numRunningDrivers_' is cleared here so that this is 0 right
    // after terminate as tests expect.
    numRunningDrivers_ = 0;
    // 遍历所有driver, 如果这个driver没有正在执行, 就将其加入到offThreadDrivers中
    for (auto& driver : drivers_) {
      if (driver) {
        // 如果driver正在执行 则这个函数返回kAlreadyOnThread
        // 否则driver没有执行 则这个函数返回kTerminate 在下面会显式调用closeByTask
        // 另外一个重要作用就是会把driver的ThreadState置上isTerminated
        if (enterForTerminateLocked(driver->state()) ==
            StopReason::kTerminate) {
          // 注意这里实际是把driver置为了nullptr 也就是外界不再能通过Task找到这个Driver
          offThreadDrivers.push_back(std::move(driver));
          driverClosedLocked();
        }
      }
    }
    exchangeClients.swap(exchangeClients_);
  }

  // 通知
  completionNotifier.notify();

  // 停掉所有off thread driver
  // Get the stats and free the resources of Drivers that were not on
  // thread.
  for (auto& driver : offThreadDrivers) {
    driver->closeByTask();
  }

  // 清理其他资源
  // ...

  // 最后返回一个terminate的future, 当所有Driver都不再执行时, future会被唤醒
  return makeFinishFuture("Task::terminate");
}

```

而`Driver::closeByTask`部分的代码就很简单了，和`Driver::close`的唯一区别就是由于这个`Driver`已经在`Task::terminate`中被置为`nullptr`，所以不需要再调用`Task::removeDriver`了。

```cpp
void Driver::closeByTask() {
  VELOX_CHECK(isTerminated());
  addStatsToTask();
  for (auto& op : operators_) {
    op->close();
  }
  closed_ = true;
}

```

### Driver的析构

上面繁杂的过程的作用都是在描述`Driver`退出的过程中，会把`Task -> Driver`的依赖消除。而`Driver -> Task`的依赖就比较简单，在`Driver`析构时，会析构`DriverCtx`，以及`DriverCtx`中的`shared_ptr<Task>`。所以`Driver -> Task`的依赖是在每个`Driver`的析构中完成，当所有一个`Task`的`Driver`都析构之后，整个`Task`也会析构。通过这样的退出方式，解决了`Task`和`Driver`互相持有对方`shared_ptr`的循环依赖问题。

从这里我们可以引申出另外一个velox退出机制中的关键点, 由于`Task::terminate`中只是停掉了一部分`Driver`，而还有一部分`Driver`持有`Task`的指针且仍在执行，这些`Driver`随时可能会调用`Task`的借口。`Task`的生命周期必须比`Driver`长，而每个`Driver`唯一持有对应的`Operator`，概括起来就是Task must outlives Driver and Operator。

### TLDR

整个velox的运行和退出机制都很复杂，牵扯到的代码细节很多，但对于退出来说，其关键点就在于这两点：

- 在`Task::terminate`函数中都只会由`Task`关掉off-thread的`Driver`，而其他on-thread或者是blocked的`Driver`会由`Driver`自身进行关闭
- `Task`的生命周期必须比`Driver`和`Operator`长