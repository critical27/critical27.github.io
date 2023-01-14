---
layout: single
title: fbthrift start-up and tear-down
date: 2022-03-05 00:00:00 +0800
categories: 实践
tags: C++ fbthrift
---

最近看了下在退出服务时在一些极端case下出现core的问题，顺便也重新梳理下fbthrift的启动和停止流程。在我们的一个服务进程中，会启动多个ThriftServer，他们的IO线程池和工作线程池是共享的，除了RPC Server对外暴露接口之外，当接收到一个RPC请求时，可能会调用服务进程中的其他组件（比如kv引擎等），这也就要求我们在退出时，需要考虑线程池共享，以及各个组件之间停止和析构的顺序。否则就会出现比如在退出服务时组件先析构，而RPC还在工作又调用了相关组件，然后core的问题。

这篇文章不会详细介绍fbthrift的相关核心概念，可以参考这个系列[文章](https://www.avabodh.com/thrift/internals.html)。

## start-up

启动涉及的接口主要就是这两个接口：

* serve
* setup

### serve

`serve`主要就是调用`setup`，然后启动一个`EventBase`将线程挂起等待退出。另外当`EventBase`退出的时候，`serve`会调用`cleanUp`清理。这里先给出一个principle

> `serve`和`stop`配对调用，`setup`和`cleanUp`配对调用

```c++
void ThriftServer::serve() {
  setup();
  if (serverChannel_ != nullptr) {
    // A duplex server (the one running on a client) doesn't uses its own EB
    // since it reuses the client's EB
    return;
  }
  SCOPE_EXIT { this->cleanUp(); };

  auto sslContextConfigCallbackHandle = sslContextObserver_
      ? getSSLCallbackHandle()
      : folly::observer::CallbackHandle{};

  tracker_.emplace(instrumentation::kThriftServerTrackerKey, *this);
  eventBaseManager_->getEventBase()->loopForever();
}
```

### setup

会对一些关键代码加一些注释，不重要的大块代码被省略。

这里面主要就是启动两个线程池`acceptPool`和`ioThreadPool`，`acceptPool`用来建立TCP链接，而`ioThreadPool`用来响应客户端的事件，比如调用read/write来读写tcp buffer，解析是需要调用哪个RPC接口，不做任何具体业务处理。`ioThreadPool`会将解析出来的RPC Request交给工作线程处理。

```c++
void ThriftServer::setup() {
  ensureDecoratedProcessorFactoryInitialized();

  auto nWorkers = getNumIOWorkerThreads();
  DCHECK_GT(nWorkers, 0u);

  addRoutingHandler(
      std::make_unique<apache::thrift::RocketRoutingHandler>(*this));

  // Initialize event base for this thread
  auto serveEventBase = eventBaseManager_->getEventBase();
  serveEventBase_ = serveEventBase;
  if (idleServerTimeout_.count() > 0) {
    idleServer_.emplace(*this, serveEventBase->timer(), idleServerTimeout_);
  }
  // Print some libevent stats
  VLOG(1) << "libevent " << folly::EventBase::getLibeventVersion() << " method "
          << serveEventBase->getLibeventMethod();

  try {
    ...

    // 设置ThriftServer的Observer 主要用来监控每个请求的排队时间 处理时间等
    if (!getObserver() && server::observerFactory_) {
      setObserver(server::observerFactory_->getObserver());
    }

    // We always need a threadmanager for cpp2.
    // 如果没有通过setThreadManager指定工作线程 会启动一个工作线程池
    setupThreadManager();
    ...

    // serverChannel_是对启动和停止使用startDuplex和stopDuplex时使用的
    // 正常的ThriftServerd都是走if路径
    if (!serverChannel_) {
      // 设置tcp链接的相关参数
      ServerBootstrap::socketConfig.acceptBacklog = getListenBacklog();
      ServerBootstrap::socketConfig.maxNumPendingConnectionsPerWorker =
          getMaxNumPendingConnectionsPerWorker();
      if (reusePort_.value_or(false)) {
        ServerBootstrap::setReusePort(true);
      }
      if (enableTFO_) {
        ServerBootstrap::socketConfig.enableTCPFastOpen = *enableTFO_;
        ServerBootstrap::socketConfig.fastOpenQueueSize = fastOpenQueueSize_;
      }

      ioThreadPool_->addObserver(
          folly::IOThreadPoolDeadlockDetectorObserver::create(
              ioThreadPool_->getName()));

      // Resize the IO pool
      ioThreadPool_->setNumThreads(nWorkers);
      // 启动accept线程池
      if (!acceptPool_) {
        acceptPool_ = std::make_shared<folly::IOThreadPoolExecutor>(
            nAcceptors_,
            std::make_shared<folly::NamedThreadFactory>("Acceptor Thread"));
      }

      auto acceptorFactory = acceptorFactory_
          ? acceptorFactory_
          : std::make_shared<DefaultThriftAcceptorFactory>(this);
      if (auto factory = dynamic_cast<wangle::AcceptorFactorySharedSSLContext*>(
              acceptorFactory.get())) {
        sharedSSLContextManager_ = factory->initSharedSSLContextManager();
      }
      ServerBootstrap::childHandler(std::move(acceptorFactory));

      // 调用wangle的接口 启动acceptPool和ioThreadPool
      {
        std::lock_guard<std::mutex> lock(ioGroupMutex_);
        ServerBootstrap::group(acceptPool_, ioThreadPool_);
      }
      if (socket_) {
        ServerBootstrap::bind(std::move(socket_));
      } else if (!getAddress().isInitialized()) {
        ServerBootstrap::bind(port_);
      } else {
        for (auto& address : addresses_) {
          ServerBootstrap::bind(address);
        }
      }
      // Update address_ with the address that we are actually bound to.
      // (This is needed if we were supplied a pre-bound socket, or if
      // address_'s port was set to 0, so an ephemeral port was chosen by
      // the kernel.)
      ServerBootstrap::getSockets()[0]->getAddress(&addresses_.at(0));

      // we enable zerocopy for the server socket if the
      // zeroCopyEnableFunc_ is valid
      ...
    } else {
      startDuplex();
    }

#if FOLLY_HAS_COROUTINES
    asyncScope_ = std::make_unique<folly::coro::CancellableAsyncScope>();
#endif
    for (auto handler : collectServiceHandlers()) {
      handler->setServer(this);
    }

    DCHECK(
        internalStatus_.load(std::memory_order_relaxed) ==
        ServerStatus::NOT_RUNNING);
    // The server is not yet ready for the user's service methods but fine to
    // handle internal methods. See ServerConfigs::getInternalMethods().
    internalStatus_.store(
        ServerStatus::PRE_STARTING, std::memory_order_release);

    // Notify handler of the preStart event
    for (const auto& eventHandler : getEventHandlersUnsafe()) {
      eventHandler->preStart(&addresses_.at(0));
    }

    internalStatus_.store(ServerStatus::STARTING, std::memory_order_release);

    // Called after setup
    callOnStartServing();

    // After the onStartServing hooks have finished, we are ready to handle
    // requests, at least from the server's perspective.
    internalStatus_.store(ServerStatus::RUNNING, std::memory_order_release);

#if FOLLY_HAS_COROUTINES
    // Set up polling for PolledServiceHealth handlers if necessary
    ...
#endif

    // Notify handler of the preServe event
    for (const auto& eventHandler : getEventHandlersUnsafe()) {
      eventHandler->preServe(&addresses_.at(0));
    }

    // Do not allow setters to be called past this point until the IO worker
    // threads have been joined in stopWorkers().
    configMutable_ = false;
  } catch (std::exception& ex) {
    // This block allows us to investigate the exception using gdb
    LOG(ERROR) << "Got an exception while setting up the server: " << ex.what();
    handleSetupFailure();
    throw;
  } catch (...) {
    handleSetupFailure();
    throw;
  }

  THRIFT_SERVER_EVENT(serve).log(*this);
}
```

### tear-down

退出时候涉及的主要接口还有析构函数

* stop
* cleanUp
* ~ThriftServer()

退出的时候主要就是调用几个函数(`stopWorkers`, `stopCPUWorkers`, `stopAcceptingAndJoinOutstandingRequests`)，逻辑比较混乱，和几个参数也有关系，后面也会画个图：

* `stopWorkersOnStopListening_` (default is true)
* `joinRequestsWhenServerStops_` (default is true)

我们先看在退出时调用的这几个函数的作用，然后再来梳理这几个函数在哪调用的。

1. `stopWorkers`

作用就是停掉`acceptPool`和`ioThreadPool_`，`ServerBootstrap`实现在`wangle`里面

```c++
void ThriftServer::stopWorkers() {
  if (serverChannel_) {
    return;
  }
  DCHECK(!duplexWorker_);
  // 不再listen socket
  ServerBootstrap::stop();
  // 调用在启动时设置的acceptPool_和ioThreadPool_的join
  ServerBootstrap::join();
  // configMutable_为true之后就可以重新setup
  configMutable_ = true;
}
```

这两个线程池都是`IOThreadPoolExecutor`，最终都是调用`ThreadPoolExecutor::join`。这里需要注意的是只会把当前`threadList_`里的线程取出来，然后调用`stopThreads`，并把它们从`threadList_`里面移除。所以并不能防止在调用`join`之后，再启动新的线程。

```c++
void ThreadPoolExecutor::join() {
  joinKeepAliveOnce();
  size_t n = 0;
  {
    SharedMutex::WriteHolder w{&threadListLock_};
    maxThreads_.store(0, std::memory_order_release);
    activeThreads_.store(0, std::memory_order_release);
    n = threadList_.get().size();
    removeThreads(n, true);
    n += threadsToJoin_.load(std::memory_order_relaxed);
    threadsToJoin_.store(0, std::memory_order_relaxed);
  }
  joinStoppedThreads(n);
  CHECK_EQ(0, threadList_.get().size());
  CHECK_EQ(0, stoppedThreads_.size());
}

void ThreadPoolExecutor::removeThreads(size_t n, bool isJoin) {
  isJoin_ = isJoin;
  stopThreads(n);
}

void IOThreadPoolExecutor::stopThreads(size_t n) {
  std::vector<ThreadPtr> stoppedThreads;
  stoppedThreads.reserve(n);
  for (size_t i = 0; i < n; i++) {
    const auto ioThread =
        std::static_pointer_cast<IOThread>(threadList_.get()[i]);
    for (auto& o : observers_) {
      o->threadStopped(ioThread.get());
    }
    ioThread->shouldRun = false;
    stoppedThreads.push_back(ioThread);
    std::lock_guard<std::mutex> guard(ioThread->eventBaseShutdownMutex_);
    if (ioThread->eventBase) {
      ioThread->eventBase->terminateLoopSoon();
    }
  }
  for (const auto& thread : stoppedThreads) {
    stoppedThreads_.add(thread);
    threadList_.remove(thread);
  }
}
```

2. `stopCPUWorkers`

主要作用就是对所有workerThread调用join，这里说的是`ThreadManager::Impl::Worker`所在线程。

```c++
void ThriftServer::stopCPUWorkers() {
  // Wait for any tasks currently running on the task queue workers to
  // finish, then stop the task queue workers. Have to do this now, so
  // there aren't tasks completing and trying to write to i/o thread
  // workers after we've stopped the i/o workers.
  threadManager_->join();
#if FOLLY_HAS_COROUTINES
  // Wait for tasks running on AsyncScope to join
  folly::coro::blockingWait(asyncScope_->joinAsync());
  cachedServiceHealth_.store(ServiceHealth{}, std::memory_order_relaxed);
#endif

  // Notify handler of the postStop event
  for (const auto& eventHandler : getEventHandlersUnsafe()) {
    eventHandler->postStop();
  }

#if FOLLY_HAS_COROUTINES
  asyncScope_.reset();
#endif
}
```

这里最关键的就是调用了`threadManager_->join();`，我们代码里使用的是`PriorityThreadManager`，大概调用链路如下：

`PriorityThreadManager::PriorityImpl::join()` -> `ThreadManager::Impl::join()` -> `ThreadManager::Impl::stopImpl`

作用就是确保每个`ThreadManager::Impl::Worker`所在的工作线程都退出，详细分析在[这里](ThreadManager.md)

3. `stopAcceptingAndJoinOutstandingRequests`

调用`AsyncServerSocket::stopAccepting`，关掉所有socket并join所有`Cpp2Worker`线程。

```c++
void ThriftServer::stopAcceptingAndJoinOutstandingRequests() {
  {
    auto expected = ServerStatus::PRE_STOPPING;
    if (!internalStatus_.compare_exchange_strong(
            expected,
            ServerStatus::STOPPING,
            std::memory_order_release,
            std::memory_order_relaxed)) {
      // stopListening() was called earlier
      DCHECK(
          expected == ServerStatus::STOPPING ||
          expected == ServerStatus::DRAINING_UNTIL_STOPPED ||
          expected == ServerStatus::NOT_RUNNING);
      return;
    }
  }

  callOnStopServing();

  internalStatus_.store(
      ServerStatus::DRAINING_UNTIL_STOPPED, std::memory_order_release);

  forEachWorker([&](wangle::Acceptor* acceptor) {
    if (auto worker = dynamic_cast<Cpp2Worker*>(acceptor)) {
      worker->requestStop();
    }
  });
  // tlsCredWatcher_ uses a background thread that needs to be joined prior
  // to any further writes to ThriftServer members.
  tlsCredWatcher_.withWLock([](auto& credWatcher) { credWatcher.reset(); });
  sharedSSLContextManager_ = nullptr;

  {
    auto sockets = getSockets();
    folly::Baton<> done;
    SCOPE_EXIT { done.wait(); };
    std::shared_ptr<folly::Baton<>> doneGuard(
        &done, [](folly::Baton<>* done) { done->post(); });

    for (auto& socket : sockets) {
      // We should have already paused accepting new connections. This just
      // closes the sockets once and for all.
      auto eb = socket->getEventBase();
      eb->runInEventBaseThread([socket = std::move(socket), doneGuard] {
        // This will also cause the workers to stop
        socket->stopAccepting();
      });
    }
  }

  auto joinDeadline =
      std::chrono::steady_clock::now() + getWorkersJoinTimeout();
  bool dumpSnapshotFlag = THRIFT_FLAG(dump_snapshot_on_long_shutdown);

  forEachWorker([&](wangle::Acceptor* acceptor) {
    if (auto worker = dynamic_cast<Cpp2Worker*>(acceptor)) {
      if (!worker->waitForStop(joinDeadline)) {
        // ...
        // 指定时间之内有Cpp2Worker没有退出 会dump一些信息然后退出
      }
    }
  });

  // Clear the decorated processor factory so that it's re-created if the server
  // is restarted.
  // Note that duplex servers drain connections in the destructor so we need to
  // keep the AsyncProcessorFactory alive until then. Duplex servers also don't
  // support restarting the server so extending its lifetime should not cause
  // issues.
  if (!isDuplex()) {
    decoratedProcessorFactory_.reset();
  }

  internalStatus_.store(ServerStatus::NOT_RUNNING, std::memory_order_release);
}
```

### stop

就是把`setup`时启动的EventBaset停掉

```c++
void ThriftServer::stop() {
  folly::EventBase* eventBase = serveEventBase_;
  if (eventBase) {
    eventBase->terminateLoopSoon();
  }
}
```

### cleanUp

在cleanUp的整个流程中`stopWorkers`和`stopAcceptingAndJoinOutstandingRequests`这两个函数会根据那几个flag的不同，调用路径会异常复杂，下面会画一张图，具体代码里就不多做阐述了。

```c++
void ThriftServer::cleanUp() {
  DCHECK(!serverChannel_);

  // tlsCredWatcher_ uses a background thread that needs to be joined prior
  // to any further writes to ThriftServer members.
  tlsCredWatcher_.withWLock([](auto& credWatcher) { credWatcher.reset(); });

  // It is users duty to make sure that setup() call
  // should have returned before doing this cleanup
  idleServer_.reset();
  serveEventBase_ = nullptr;
  stopListening();

  // Stop the routing handlers.
  for (auto& handler : routingHandlers_) {
    handler->stopListening();
  }

  if (stopWorkersOnStopListening_) {
    // Wait on the i/o worker threads to actually stop
    stopWorkers();
  } else if (joinRequestsWhenServerStops_) {
    stopAcceptingAndJoinOutstandingRequests();
  }

  for (auto handler : getProcessorFactory()->getServiceHandlers()) {
    handler->setServer(nullptr);
  }

  // Now clear all the handlers
  routingHandlers_.clear();
}
```

### ~ThriftServer()

```c++
ThriftServer::~ThriftServer() {
  tracker_.reset();
  if (duplexWorker_) {
    // usually ServerBootstrap::stop drains the workers, but ServerBootstrap
    // doesn't know about duplexWorker_
    duplexWorker_->drainAllConnections();

    LOG_IF(ERROR, !duplexWorker_.unique())
        << getActiveRequests() << " active Requests while in destructing"
        << " duplex ThriftServer. Consider using startDuplex & stopDuplex";
  }

  if (stopWorkersOnStopListening_) {
    // Everything is already taken care of.
    return;
  }
  // If the flag is false, neither i/o nor CPU workers are stopped at this
  // point. Stop them now.
  if (!joinRequestsWhenServerStops_) {
    stopAcceptingAndJoinOutstandingRequests();
  }
  stopCPUWorkers();
  stopWorkers();
}
```

`stopListening`就是调用`AsyncServerSocket::pauseAccepting`把所有socket绑定的所有`EventHandler`都unregister掉(不会关掉socket 而`AsyncServerSocket::stopAccepting`除了unregister所有`EventHandler`还会把所有socket都关掉)，也就不再处理任何事件。

```c++
void ThriftServer::stopListening() {
  // We have to make sure stopListening() is not called twice when both
  // stopListening() and cleanUp() are called
  {
    auto expected = ServerStatus::RUNNING;
    if (!internalStatus_.compare_exchange_strong(
            expected,
            ServerStatus::PRE_STOPPING,
            std::memory_order_release,
            std::memory_order_relaxed)) {
      // stopListening() was called earlier
      DCHECK(
          expected == ServerStatus::PRE_STOPPING ||
          expected == ServerStatus::STOPPING ||
          expected == ServerStatus::DRAINING_UNTIL_STOPPED ||
          expected == ServerStatus::NOT_RUNNING);
      return;
    }
  }

#if FOLLY_HAS_COROUTINES
  asyncScope_->requestCancellation();
#endif

  {
    auto sockets = getSockets();
    folly::Baton<> done;
    SCOPE_EXIT { done.wait(); };
    std::shared_ptr<folly::Baton<>> doneGuard(
        &done, [](folly::Baton<>* done) { done->post(); });

    for (auto& socket : sockets) {
      // Stop accepting new connections
      auto eb = socket->getEventBase();
      eb->runInEventBaseThread([socket = std::move(socket), doneGuard] {
        socket->pauseAccepting();
      });
    }
  }

  if (stopWorkersOnStopListening_) {
    stopAcceptingAndJoinOutstandingRequests();
    stopCPUWorkers();
  }
}
```

### 调用关系

我们把退出时候的几个接口和要join的线程整理如下图所示，黑线是无条件调用，带颜色的线都是条件成立再调用。

![figure]({{'/archive/ThriftServerStop.png' | prepend: site.baseurl}})

我们分几种情况来说明，由于`cleanUp`和析构可能都会调用这些函数，所以有两个分支。`cleanUp`都会在析构之前调用。

1. `stopWorkersOnStopListening_` = true, `joinRequestsWhenServerStops_` = true, 这是默认值

调用顺序：

```
  `cleanUp` -> `stopListening` -> `stopAcceptingAndJoinOutstandingRequests` -> `stopCPUWorkers` -> `stopWorkers`

  `~ThriftServer` -> `stopCPUWorkers` -> `stopWorkers`
```

* `stopWorkersOnStopListening_` = false, `joinRequestsWhenServerStops_` = true

调用顺序：

```
  `cleanUp` -> `stopListening` -> `stopAcceptingAndJoinOutstandingRequests`

  `~ThriftServer` -> `stopCPUWorkers` -> `stopWorkers`
```

* `stopWorkersOnStopListening_` = true, `joinRequestsWhenServerStops_` = false

调用顺序：

```
  `cleanUp` -> `stopListening` -> `stopAcceptingAndJoinOutstandingRequests` -> `stopCPUWorkers` -> `stopWorkers`

  `~ThriftServer` -> `stopAcceptingAndJoinOutstandingRequests` -> `stopCPUWorkers` -> `stopWorkers`
```

* `stopWorkersOnStopListening_` = false, `joinRequestsWhenServerStops_` = false

调用顺序：

```
  `cleanUp` -> `stopListening`

  `~ThriftServer` -> `stopAcceptingAndJoinOutstandingRequests` -> `stopCPUWorkers` -> `stopWorkers`
```

---

结合这四种情况，我们可以看到`stopWorkers`, `stopCPUWorkers`, `stopAcceptingAndJoinOutstandingRequests`这三个函数可能会调用多次，由于`cleanUp`一定在析构之前调用，所以这三个函数的第一次调用一定满足如下条件：

`stopAcceptingAndJoinOutstandingRequests` -> `stopCPUWorkers` -> `stopWorkers`

也就是在ThriftServer退出或者析构的时候，一定会先关闭socket并join所有`Cpp2Worker`，然后join所有`ThreadManger`的`Worker`，最后再关闭`acceptPool`和`ioThreadPool`。

Done~
