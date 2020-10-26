# **Server method**

## Start 方法

<details> 

<summary>start 方法</summary>

```
void Server::Start(grpc::ServerCompletionQueue** cqs, size_t num_cqs) {
  GPR_ASSERT(!started_);
  global_callbacks_->PreServerStart(this);
  started_ = true;

  // Only create default health check service when user did not provide an
  // explicit one.
  grpc::ServerCompletionQueue* health_check_cq = nullptr;
  grpc::DefaultHealthCheckService::HealthCheckServiceImpl*
      default_health_check_service_impl = nullptr;
  if (health_check_service_ == nullptr && !health_check_service_disabled_ &&
      grpc::DefaultHealthCheckServiceEnabled()) {
    auto* default_hc_service = new grpc::DefaultHealthCheckService;
    health_check_service_.reset(default_hc_service);
    // We create a non-polling CQ to avoid impacting application
    // performance.  This ensures that we don't introduce thread hops
    // for application requests that wind up on this CQ, which is polled
    // in its own thread.
    health_check_cq = new grpc::ServerCompletionQueue(
        GRPC_CQ_NEXT, GRPC_CQ_NON_POLLING, nullptr);
    grpc_server_register_completion_queue(server_, health_check_cq->cq(),
                                          nullptr);
    default_health_check_service_impl =
        default_hc_service->GetHealthCheckService(
            std::unique_ptr<grpc::ServerCompletionQueue>(health_check_cq));
    RegisterService(nullptr, default_health_check_service_impl);
  }

  for (auto& acceptor : acceptors_) {
    acceptor->GetCredentials()->AddPortToServer(acceptor->name(), server_);
  }

  // If this server uses callback methods, then create a callback generic
  // service to handle any unimplemented methods using the default reactor
  // creator
  if (has_callback_methods_ && !has_callback_generic_service_) {
    unimplemented_service_ = absl::make_unique<grpc::CallbackGenericService>();
    RegisterCallbackGenericService(unimplemented_service_.get());
  }

#ifndef NDEBUG
  for (size_t i = 0; i < num_cqs; i++) {
    cq_list_.push_back(cqs[i]);
  }
#endif

  grpc_server_start(server_);

  if (!has_async_generic_service_ && !has_callback_generic_service_) {
    for (const auto& value : sync_req_mgrs_) {
      value->AddUnknownSyncMethod();
    }

    for (size_t i = 0; i < num_cqs; i++) {
      if (cqs[i]->IsFrequentlyPolled()) {
        new UnimplementedAsyncRequest(this, cqs[i]);
      }
    }
    if (health_check_cq != nullptr) {
      new UnimplementedAsyncRequest(this, health_check_cq);
    }
  }

  // If this server has any support for synchronous methods (has any sync
  // server CQs), make sure that we have a ResourceExhausted handler
  // to deal with the case of thread exhaustion
  if (sync_server_cqs_ != nullptr && !sync_server_cqs_->empty()) {
    resource_exhausted_handler_ =
        absl::make_unique<grpc::internal::ResourceExhaustedHandler>();
  }

  for (const auto& value : sync_req_mgrs_) {
    value->Start();
  }

  if (default_health_check_service_impl != nullptr) {
    default_health_check_service_impl->StartServingThread();
  }

  for (auto& acceptor : acceptors_) {
    acceptor->Start();
  }
}
```

</details>