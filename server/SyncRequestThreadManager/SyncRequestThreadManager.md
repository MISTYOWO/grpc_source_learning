# SyncRequestThreadManager

## 组件

```
  Server* server_;
  grpc::CompletionQueue* server_cq_;
  int cq_timeout_msec_;
  std::vector<std::unique_ptr<SyncRequest>> sync_requests_;
  std::unique_ptr<grpc::internal::RpcServiceMethod> unknown_method_;
  std::shared_ptr<Server::GlobalCallbacks> global_callbacks_;
```

## 核心方法

<details>

<summary>
start()逻辑
</summary>

```
start()逻辑
/*
    遍历sync_requests_
    调用继承ThreadManager的初始化方法
*/
void Start() {
    if (!sync_requests_.empty()) {
      for (const auto& value : sync_requests_) {
        value->SetupRequest(); // completion_queue创建
        value->Request(server_->c_server(), server_cq_->cq());
      }
      Initialize();  // ThreadManager's Initialize()
    }
  }
```

SyncRequest Request()逻辑
```
/*
    根据是否注册过，调用对应的Call方法
*/
void Request(grpc_server* server, grpc_completion_queue* notify_cq) {
    GPR_ASSERT(cq_ && !in_flight_);
    in_flight_ = true;
    // 注册的method_tag_的方法指针
    if (method_tag_) {
      if (grpc_server_request_registered_call(
              server, method_tag_, &call_, &deadline_, &request_metadata_,
              has_request_payload_ ? &request_payload_ : nullptr, cq_,
              notify_cq, this) != GRPC_CALL_OK) {
        TeardownRequest();
        return;
      }
    } else {
      if (!call_details_) {
        call_details_ = new grpc_call_details;
        grpc_call_details_init(call_details_);
      }
      if (grpc_server_request_call(server, &call_, call_details_,
                                   &request_metadata_, cq_, notify_cq,
                                   this) != GRPC_CALL_OK) {
        TeardownRequest();
        return;
      }
    }
  }
```

如果是注册过的method逻辑走 ，method_tag通过static_cast 转化成RegisteredMethod* rm
```
grpc_call_error Server::RequestRegisteredCall(
    RegisteredMethod* rm, grpc_call** call, gpr_timespec* deadline,
    grpc_metadata_array* request_metadata, grpc_byte_buffer** optional_payload,
    grpc_completion_queue* cq_bound_to_call,
    grpc_completion_queue* cq_for_notification, void* tag_new) {
  size_t cq_idx;

  // 初始化了cq_idx
  grpc_call_error error = ValidateServerRequestAndCq(
      &cq_idx, cq_for_notification, tag_new, optional_payload, rm);
  if (error != GRPC_CALL_OK) {
    return error;
  }
  RequestedCall* rc =
      new RequestedCall(tag_new, cq_bound_to_call, call, request_metadata, rm,
                        deadline, optional_payload);
  return QueueRequestedCall(cq_idx, rc);
}
```
基于rc的type是否注册过，获取不同的RequestMatcherInterface进行调用
```
grpc_call_error Server::QueueRequestedCall(size_t cq_idx, RequestedCall* rc) {
  if (shutdown_flag_.load(std::memory_order_acquire)) {
    FailCall(cq_idx, rc,
             GRPC_ERROR_CREATE_FROM_STATIC_STRING("Server Shutdown"));
    return GRPC_CALL_OK;
  }
  RequestMatcherInterface* rm;
  switch (rc->type) {
    case RequestedCall::Type::BATCH_CALL:
      rm = unregistered_request_matcher_.get();
      break;
    case RequestedCall::Type::REGISTERED_CALL:
      rm = rc->data.registered.method->matcher.get();
      break;
  }
  rm->RequestCallWithPossiblePublish(cq_idx, rc);
  return GRPC_CALL_OK;
}
```

最后根据 request_queue_index 和 RequestedCall 调用 CallData.Publish() (用于第一次匹配上的调用) 
```
    void RequestCallWithPossiblePublish(size_t request_queue_index,
                                      RequestedCall* call) override {
    if (requests_per_cq_[request_queue_index].Push(&call->mpscq_node)) {
      /* this was the first queued request: we need to lock and start
         matching calls */
      struct PendingCall {
        RequestedCall* rc = nullptr;
        CallData* calld;
      };
      auto pop_next_pending = [this, request_queue_index] {
        PendingCall pending_call;
        {
          MutexLock lock(&server_->mu_call_);
          if (!pending_.empty()) {
            pending_call.rc = reinterpret_cast<RequestedCall*>(
                requests_per_cq_[request_queue_index].Pop());
            if (pending_call.rc != nullptr) {
              pending_call.calld = pending_.front();
              pending_.pop();
            }
          }
        }
        return pending_call;
      };
      while (true) {
        PendingCall next_pending = pop_next_pending();
        if (next_pending.rc == nullptr) break;
        if (!next_pending.calld->MaybeActivate()) {
          // Zombied Call
          next_pending.calld->KillZombie();
        } else {
          next_pending.calld->Publish(request_queue_index, next_pending.rc);
        }
      }
    }
  }
```
将RequestedCall和Server::DoneRequestEvent函数指针传入cq_idx对应的grpc_cq_end_op方法，最后由vtable负责调用
```
void Server::CallData::Publish(size_t cq_idx, RequestedCall* rc) {
  grpc_call_set_completion_queue(call_, rc->cq_bound_to_call);
  *rc->call = call_;
  cq_new_ = server_->cqs_[cq_idx];
  GPR_SWAP(grpc_metadata_array, *rc->initial_metadata, initial_metadata_);
  switch (rc->type) {
    case RequestedCall::Type::BATCH_CALL:
      GPR_ASSERT(host_.has_value());
      GPR_ASSERT(path_.has_value());
      rc->data.batch.details->host = grpc_slice_ref_internal(*host_);
      rc->data.batch.details->method = grpc_slice_ref_internal(*path_);
      rc->data.batch.details->deadline =
          grpc_millis_to_timespec(deadline_, GPR_CLOCK_MONOTONIC);
      rc->data.batch.details->flags = recv_initial_metadata_flags_;
      break;
    case RequestedCall::Type::REGISTERED_CALL:
      *rc->data.registered.deadline =
          grpc_millis_to_timespec(deadline_, GPR_CLOCK_MONOTONIC);
      if (rc->data.registered.optional_payload != nullptr) {
        *rc->data.registered.optional_payload = payload_;
        payload_ = nullptr;
      }
      break;
    default:
      GPR_UNREACHABLE_CODE(return );
  }
  grpc_cq_end_op(cq_new_, rc->tag, GRPC_ERROR_NONE, Server::DoneRequestEvent,
                 rc, &rc->completion, true);
}
```
grpc_completion_queue的vtable负责处理
```
void grpc_cq_end_op(grpc_completion_queue* cq, void* tag, grpc_error* error,
                    void (*done)(void* done_arg, grpc_cq_completion* storage),
                    void* done_arg, grpc_cq_completion* storage,
                    bool internal) {
  cq->vtable->end_op(cq, tag, error, done, done_arg, storage, internal);
}
```
vtable 对应三种 vtable，根据next，callback和pluck三种

```
static const cq_vtable g_cq_vtable[] = {
    /* GRPC_CQ_NEXT */
    {GRPC_CQ_NEXT, sizeof(cq_next_data), cq_init_next, cq_shutdown_next,
     cq_destroy_next, cq_begin_op_for_next, cq_end_op_for_next, cq_next,
     nullptr},
    /* GRPC_CQ_PLUCK */
    {GRPC_CQ_PLUCK, sizeof(cq_pluck_data), cq_init_pluck, cq_shutdown_pluck,
     cq_destroy_pluck, cq_begin_op_for_pluck, cq_end_op_for_pluck, nullptr,
     cq_pluck},
    /* GRPC_CQ_CALLBACK */
    {GRPC_CQ_CALLBACK, sizeof(cq_callback_data), cq_init_callback,
     cq_shutdown_callback, cq_destroy_callback, cq_begin_op_for_callback,
     cq_end_op_for_callback, nullptr, nullptr},
};
```

next 对应end_op
封装成grpc_cq_completion，压入到 cq_next_data的queue中
```
static void cq_end_op_for_next(
    grpc_completion_queue* cq, void* tag, grpc_error* error,
    void (*done)(void* done_arg, grpc_cq_completion* storage), void* done_arg,
    grpc_cq_completion* storage, bool /*internal*/) {
  GPR_TIMER_SCOPE("cq_end_op_for_next", 0);

  if (GRPC_TRACE_FLAG_ENABLED(grpc_api_trace) ||
      (GRPC_TRACE_FLAG_ENABLED(grpc_trace_operation_failures) &&
       error != GRPC_ERROR_NONE)) {
    const char* errmsg = grpc_error_string(error);
    GRPC_API_TRACE(
        "cq_end_op_for_next(cq=%p, tag=%p, error=%s, "
        "done=%p, done_arg=%p, storage=%p)",
        6, (cq, tag, errmsg, done, done_arg, storage));
    if (GRPC_TRACE_FLAG_ENABLED(grpc_trace_operation_failures) &&
        error != GRPC_ERROR_NONE) {
      gpr_log(GPR_ERROR, "Operation failed: tag=%p, error=%s", tag, errmsg);
    }
  }
  cq_next_data* cqd = static_cast<cq_next_data*> DATA_FROM_CQ(cq);
  int is_success = (error == GRPC_ERROR_NONE);

  storage->tag = tag;
  storage->done = done;
  storage->done_arg = done_arg;
  storage->next = static_cast<uintptr_t>(is_success);

  cq_check_tag(cq, tag, true); /* Used in debug builds only */

  if ((grpc_completion_queue*)gpr_tls_get(&g_cached_cq) == cq &&
      (grpc_cq_completion*)gpr_tls_get(&g_cached_event) == nullptr) {
    gpr_tls_set(&g_cached_event, (intptr_t)storage);
  } else {
    /* Add the completion to the queue */
    bool is_first = cqd->queue.Push(storage);
    cqd->things_queued_ever.FetchAdd(1, grpc_core::MemoryOrder::RELAXED);
    /* Since we do not hold the cq lock here, it is important to do an 'acquire'
       load here (instead of a 'no_barrier' load) to match with the release
       store
       (done via pending_events.FetchSub(1, ACQ_REL)) in cq_shutdown_next
       */
    if (cqd->pending_events.Load(grpc_core::MemoryOrder::ACQUIRE) != 1) {
      /* Only kick if this is the first item queued */
      if (is_first) {
        gpr_mu_lock(cq->mu);
        grpc_error* kick_error =
            cq->poller_vtable->kick(POLLSET_FROM_CQ(cq), nullptr);
        gpr_mu_unlock(cq->mu);

        if (kick_error != GRPC_ERROR_NONE) {
          const char* msg = grpc_error_string(kick_error);
          gpr_log(GPR_ERROR, "Kick failed: %s", msg);
          GRPC_ERROR_UNREF(kick_error);
        }
      }
      if (cqd->pending_events.FetchSub(1, grpc_core::MemoryOrder::ACQ_REL) ==
          1) {
        GRPC_CQ_INTERNAL_REF(cq, "shutting_down");
        gpr_mu_lock(cq->mu);
        cq_finish_shutdown_next(cq);
        gpr_mu_unlock(cq->mu);
        GRPC_CQ_INTERNAL_UNREF(cq, "shutting_down");
      }
    } else {
      GRPC_CQ_INTERNAL_REF(cq, "shutting_down");
      cqd->pending_events.Store(0, grpc_core::MemoryOrder::RELEASE);
      gpr_mu_lock(cq->mu);
      cq_finish_shutdown_next(cq);
      gpr_mu_unlock(cq->mu);
      GRPC_CQ_INTERNAL_UNREF(cq, "shutting_down");
    }
  }

  GRPC_ERROR_UNREF(error);
}
```
</details>