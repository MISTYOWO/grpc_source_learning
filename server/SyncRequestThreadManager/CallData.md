# CallData

## 组件

    grpc::CompletionQueue cq_;
    grpc::ServerContext ctx_;
    const bool has_request_payload_;
    grpc_byte_buffer* request_payload_;
    void* request_;
    grpc::Status request_status_;
    grpc::internal::RpcServiceMethod* const method_;
    grpc::internal::Call call_;
    Server* server_;
    std::shared_ptr<GlobalCallbacks> global_callbacks_;
    bool resources_;
    grpc::internal::InterceptorBatchMethodsImpl interceptor_methods_;

## 核心方法

<details>

<summary>
Run()核心逻辑
</summary>

```
void Run(const std::shared_ptr<GlobalCallbacks>& global_callbacks,
             bool resources) {
      global_callbacks_ = global_callbacks;
      resources_ = resources;

      interceptor_methods_.SetCall(&call_);
      interceptor_methods_.SetReverse();
      // Set interception point for RECV INITIAL METADATA
      interceptor_methods_.AddInterceptionHookPoint(
          grpc::experimental::InterceptionHookPoints::
              POST_RECV_INITIAL_METADATA);
      interceptor_methods_.SetRecvInitialMetadata(&ctx_.client_metadata_);

      if (has_request_payload_) {
        // Set interception point for RECV MESSAGE
        auto* handler = resources_ ? method_->handler()
                                   : server_->resource_exhausted_handler_.get();
        request_ = handler->Deserialize(call_.call(), request_payload_,
                                        &request_status_, nullptr);

        request_payload_ = nullptr;
        interceptor_methods_.AddInterceptionHookPoint(
            grpc::experimental::InterceptionHookPoints::POST_RECV_MESSAGE);
        interceptor_methods_.SetRecvMessage(request_, nullptr);
      }

      if (interceptor_methods_.RunInterceptors(
              [this]() { ContinueRunAfterInterception(); })) {
                //实际调用包装的方法的地方
        ContinueRunAfterInterception();
      } else {
        // There were interceptors to be run, so ContinueRunAfterInterception
        // will be run when interceptors are done.
      }
    }
```
ContinueRunAfterInterception() 调用方法
```
    void ContinueRunAfterInterception() {
      {
        ctx_.BeginCompletionOp(&call_, nullptr, nullptr);
        global_callbacks_->PreSynchronousRequest(&ctx_);
        auto* handler = resources_ ? method_->handler()
                                   : server_->resource_exhausted_handler_.get();
        handler->RunHandler(grpc::internal::MethodHandler::HandlerParameter(
            &call_, &ctx_, request_, request_status_, nullptr, nullptr));
        request_ = nullptr;
        global_callbacks_->PostSynchronousRequest(&ctx_);

        cq_.Shutdown();

        grpc::internal::CompletionQueueTag* op_tag = ctx_.GetCompletionOpTag();
        cq_.TryPluck(op_tag, gpr_inf_future(GPR_CLOCK_REALTIME));

        /* Ensure the cq_ is shutdown */
        grpc::DummyTag ignored_tag;
        GPR_ASSERT(cq_.Pluck(&ignored_tag) == false);
      }
      delete this;
    }
```
</details>