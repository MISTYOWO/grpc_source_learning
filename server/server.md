# Server component

## server_

<details>

<summary>
grpc_server（server_封装)
</summary>

```
  grpc_channel_args* const channel_args_;
  grpc_resource_user* default_resource_user_ = nullptr;
  RefCountedPtr<channelz::ServerNode> channelz_node_;

  std::vector<grpc_completion_queue*> cqs_;
  std::vector<grpc_pollset*> pollsets_;
  bool started_ = false;

  // The two following mutexes control access to server-state.
  // mu_global_ controls access to non-call-related state (e.g., channel state).
  // mu_call_ controls access to call-related state (e.g., the call lists).
  //
  // If they are ever required to be nested, you must lock mu_global_
  // before mu_call_. This is currently used in shutdown processing
  // (ShutdownAndNotify() and MaybeFinishShutdown()).
  Mutex mu_global_;  // mutex for server and channel state
  Mutex mu_call_;    // mutex for call-specific state

  // startup synchronization: flag is protected by mu_global_, signals whether
  // we are doing the listener start routine or not.
  bool starting_ = false;
  CondVar starting_cv_;

  std::vector<std::unique_ptr<RegisteredMethod>> registered_methods_;

  // Request matcher for unregistered methods.
  std::unique_ptr<RequestMatcherInterface> unregistered_request_matcher_;

  std::atomic_bool shutdown_flag_{false};
  bool shutdown_published_ = false;
  std::vector<ShutdownTag> shutdown_tags_;

  std::list<ChannelData*> channels_;

  std::list<Listener> listeners_;
  size_t listeners_destroyed_ = 0;

  // The last time we printed a shutdown progress message.
  gpr_timespec last_shutdown_message_time_;
```
</details>



