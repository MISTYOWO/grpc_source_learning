# ThreadManager

## 主要组件
```
  grpc_core::Mutex mu_;

  bool shutdown_;
  grpc_core::CondVar shutdown_cv_;

  // The resource user object to use when requesting quota to create threads
  //
  // Note: The user of this ThreadManager object must create grpc_resource_quota
  // object (that contains the actual max thread quota) and a grpc_resource_user
  // object through which quota is requested whenever new threads need to be
  // created
  grpc_resource_user* resource_user_;

  // Number of threads doing polling
  int num_pollers_;

  // The minimum and maximum number of threads that should be doing polling
  int min_pollers_;
  int max_pollers_;

  // The total number of threads currently active (includes threads includes the
  // threads that are currently polling i.e num_pollers_)
  int num_threads_;

  // See GetMaxActiveThreadsSoFar()'s description.
  // To be more specific, this variable tracks the max value num_threads_ was
  // ever set so far
  int max_active_threads_sofar_;

  grpc_core::Mutex list_mu_;
  std::list<WorkerThread*> completed_threads_;
```


## 核心方法

<details>
<summary>
Initialize() 初始化核心组件，然后对workThread调用start方法
</summary>

```
void ThreadManager::Initialize() {
  if (!grpc_resource_user_allocate_threads(resource_user_, min_pollers_)) {
    gpr_log(GPR_ERROR,
            "No thread quota available to even create the minimum required "
            "polling threads (i.e %d). Unable to start the thread manager",
            min_pollers_);
    abort();
  }

  {
    grpc_core::MutexLock lock(&mu_);
    num_pollers_ = min_pollers_;
    num_threads_ = min_pollers_;
    max_active_threads_sofar_ = min_pollers_;
  }

  for (int i = 0; i < min_pollers_; i++) {
    WorkerThread* worker = new WorkerThread(this);
    GPR_ASSERT(worker->created());  // Must be able to create the minimum
    worker->Start();
  }
}
```

</details>

<details>
<summary>
WorkerThread(ThreadManager* thd_mgr)
</summary>

```
// 将WorkerThread -> Run的函数指针传入Thread的构造函数，然后thd_ 指向它
ThreadManager::WorkerThread::WorkerThread(ThreadManager* thd_mgr)
    : thd_mgr_(thd_mgr) {
  // Make thread creation exclusive with respect to its join happening in
  // ~WorkerThread().
  thd_ = grpc_core::Thread(
      "grpcpp_sync_server",
      [](void* th) { static_cast<ThreadManager::WorkerThread*>(th)->Run(); },
      this, &created_);
  if (!created_) {
    gpr_log(GPR_ERROR, "Could not create grpc_sync_server worker-thread");
  }
}

//就做两件事，
//1. 拉工作，然后封装成CallData，run
//2. 做完之后标记已完成
void ThreadManager::WorkerThread::Run() {
  thd_mgr_->MainWorkLoop();
  thd_mgr_->MarkAsCompleted(this);
}

// 拉取work，根据需要新增线程或者删除线程
void ThreadManager::MainWorkLoop() {
  while (true) {
    void* tag;
    bool ok;
    WorkStatus work_status = PollForWork(&tag, &ok);

    grpc_core::ReleasableMutexLock lock(&mu_);
    // Reduce the number of pollers by 1 and check what happened with the poll
    num_pollers_--;
    bool done = false;
    switch (work_status) {
      case TIMEOUT:
        // If we timed out and we have more pollers than we need (or we are
        // shutdown), finish this thread
        if (shutdown_ || num_pollers_ > max_pollers_) done = true;
        break;
      case SHUTDOWN:
        // If the thread manager is shutdown, finish this thread
        done = true;
        break;
      case WORK_FOUND:
        // If we got work and there are now insufficient pollers and there is
        // quota available to create a new thread, start a new poller thread
        bool resource_exhausted = false;
        if (!shutdown_ && num_pollers_ < min_pollers_) {
          if (grpc_resource_user_allocate_threads(resource_user_, 1)) {
            // We can allocate a new poller thread
            num_pollers_++;
            num_threads_++;
            if (num_threads_ > max_active_threads_sofar_) {
              max_active_threads_sofar_ = num_threads_;
            }
            // Drop lock before spawning thread to avoid contention
            lock.Unlock();
            WorkerThread* worker = new WorkerThread(this);
            if (worker->created()) {
              worker->Start();
            } else {
              // Get lock again to undo changes to poller/thread counters.
              grpc_core::MutexLock failure_lock(&mu_);
              num_pollers_--;
              num_threads_--;
              resource_exhausted = true;
              delete worker;
            }
          } else if (num_pollers_ > 0) {
            // There is still at least some thread polling, so we can go on
            // even though we are below the number of pollers that we would
            // like to have (min_pollers_)
            lock.Unlock();
          } else {
            // There are no pollers to spare and we couldn't allocate
            // a new thread, so resources are exhausted!
            lock.Unlock();
            resource_exhausted = true;
          }
        } else {
          // There are a sufficient number of pollers available so we can do
          // the work and continue polling with our existing poller threads
          lock.Unlock();
        }
        // Lock is always released at this point - do the application work
        // or return resource exhausted if there is new work but we couldn't
        // get a thread in which to do it.
        DoWork(tag, ok, !resource_exhausted);
        // Take the lock again to check post conditions
        lock.Lock();
        // If we're shutdown, we should finish at this point.
        if (shutdown_) done = true;
        break;
    }
    // If we decided to finish the thread, break out of the while loop
    if (done) break;

    // Otherwise go back to polling as long as it doesn't exceed max_pollers_
    //
    // **WARNING**:
    // There is a possibility of threads thrashing here (i.e excessive thread
    // shutdowns and creations than the ideal case). This happens if max_poller_
    // count is small and the rate of incoming requests is also small. In such
    // scenarios we can possibly configure max_pollers_ to a higher value and/or
    // increase the cq timeout.
    //
    // However, not doing this check here and unconditionally incrementing
    // num_pollers (and hoping that the system will eventually settle down) has
    // far worse consequences i.e huge number of threads getting created to the
    // point of thread-exhaustion. For example: if the incoming request rate is
    // very high, all the polling threads will return very quickly from
    // PollForWork() with WORK_FOUND. They all briefly decrement num_pollers_
    // counter thereby possibly - and briefly - making it go below min_pollers;
    // This will most likely result in the creation of a new poller since
    // num_pollers_ dipped below min_pollers_.
    //
    // Now, If we didn't do the max_poller_ check here, all these threads will
    // go back to doing PollForWork() and the whole cycle repeats (with a new
    // thread being added in each cycle). Once the total number of threads in
    // the system crosses a certain threshold (around ~1500), there is heavy
    // contention on mutexes (the mu_ here or the mutexes in gRPC core like the
    // pollset mutex) that makes DoWork() take longer to finish thereby causing
    // new poller threads to be created even faster. This results in a thread
    // avalanche.
    if (num_pollers_ < max_pollers_) {
      num_pollers_++;
    } else {
      break;
    }
  };

// 将work封装成CallData，然调用CallData的Run函数
void DoWork(void* tag, bool ok, bool resources) override {
    SyncRequest* sync_req = static_cast<SyncRequest*>(tag);

    if (!sync_req) {
      // No tag. Nothing to work on. This is an unlikley scenario and possibly a
      // bug in RPC Manager implementation.
      gpr_log(GPR_ERROR, "Sync server. DoWork() was called with NULL tag");
      return;
    }

    if (ok) {
      // Calldata takes ownership of the completion queue and interceptors
      // inside sync_req
      auto* cd = new SyncRequest::CallData(server_, sync_req);
      // Prepare for the next request
      if (!IsShutdown()) {
        sync_req->SetupRequest();  // Create new completion queue for sync_req
        sync_req->Request(server_->c_server(), server_cq_->cq());
      }

      GPR_TIMER_SCOPE("cd.Run()", 0);
      cd->Run(global_callbacks_, resources);
    }
    // TODO (sreek) If ok is false here (which it isn't in case of
    // grpc_request_registered_call), we should still re-queue the request
    // object
  }

// 将workThread标记为已完成
void ThreadManager::MarkAsCompleted(WorkerThread* thd) {
  {
    grpc_core::MutexLock list_lock(&list_mu_);
    completed_threads_.push_back(thd);
  }

  {
    grpc_core::MutexLock lock(&mu_);
    num_threads_--;
    if (num_threads_ == 0) {
      shutdown_cv_.Signal();
    }
  }

  // Give a thread back to the resource quota
  grpc_resource_user_free_threads(resource_user_, 1);
}
```

</details>





