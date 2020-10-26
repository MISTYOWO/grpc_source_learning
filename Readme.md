# grpc源码学习
**本项目个人阅读学习源码，持续更新，如果存在不足之处欢迎指正**
## Server 组件
```

  std::vector<std::shared_ptr<internal::ExternalConnectionAcceptorImpl>>
      acceptors_;
  // 拦截器
  std::vector<std::unique_ptr<experimental::ServerInterceptorFactoryInterface>>
      interceptor_creators_;
  // 接受的最大信息尺寸
  int max_receive_message_size_;
  //已完成队列，重复利用处理新的rpc
  std::shared_ptr<std::vector<std::unique_ptr<ServerCompletionQueue>>>
      sync_server_cqs_;
  //线程管理组件
  std::vector<std::unique_ptr<SyncRequestThreadManager>> sync_req_mgrs_;
#ifndef GRPC_CALLBACK_API_NONEXPERIMENTAL
  experimental_registration_type experimental_registration_{this};
#endif
  //
  internal::Mutex mu_;
  // server状态
  bool started_;
  bool shutdown_;
  bool shutdown_notified_;  // Was notify called on the shutdown_cv_
  internal::CondVar shutdown_done_cv_;
  bool shutdown_done_ = false;
  std::atomic_int shutdown_refs_outstanding_{1};
  internal::CondVar shutdown_cv_;
  std::shared_ptr<GlobalCallbacks> global_callbacks_;
  std::vector<std::string> services_;
  bool has_async_generic_service_ = false;
  bool has_callback_generic_service_ = false;
  bool has_callback_methods_ = false;

  //封装的server
  grpc_server* server_;
  std::unique_ptr<ServerInitializer> server_initializer_;
  std::unique_ptr<HealthCheckServiceInterface> health_check_service_;
  bool health_check_service_disabled_;
  
  //回调函数
#ifdef GRPC_CALLBACK_API_NONEXPERIMENTAL
  std::unique_ptr<CallbackGenericService> unimplemented_service_;
#else
  std::unique_ptr<experimental::CallbackGenericService> unimplemented_service_;
#endif
  //负责处理resource的handler
  std::unique_ptr<internal::MethodHandler> resource_exhausted_handler_;
  //通用处理回调函数的handler
  std::unique_ptr<internal::MethodHandler> generic_handler_;
  //已完成队列
  CompletionQueue* callback_cq_ /* GUARDED_BY(mu_) */ = nullptr;
  std::vector<CompletionQueue*> cq_list_;
  ```
