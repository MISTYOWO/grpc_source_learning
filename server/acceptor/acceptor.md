# ExternalConnectionAcceptorImpl核心组件和头文件

目前来看主要功能是处理外部连接，核心组件 TcpServerFdHandler

```
class ExternalConnectionAcceptorImpl
    : public std::enable_shared_from_this<ExternalConnectionAcceptorImpl> {
 public:
  ExternalConnectionAcceptorImpl(
      const std::string& name,
      ServerBuilder::experimental_type::ExternalConnectionType type,
      std::shared_ptr<ServerCredentials> creds);
  // Should only be called once.
  std::unique_ptr<experimental::ExternalConnectionAcceptor> GetAcceptor();

  void HandleNewConnection(
      experimental::ExternalConnectionAcceptor::NewConnectionParameters* p);

  void Shutdown();

  void Start();

  const char* name() { return name_.c_str(); }

  ServerCredentials* GetCredentials() { return creds_.get(); }

  void SetToChannelArgs(::grpc::ChannelArguments* args);

 private:
  const std::string name_;
  std::shared_ptr<ServerCredentials> creds_;
  grpc_core::TcpServerFdHandler* handler_ = nullptr;  // not owned
  grpc_core::Mutex mu_;
  bool has_acceptor_ = false;
  bool started_ = false;
  bool shutdown_ = false;
};
```

# 主要逻辑HandleNewConnection

 封装一层将新连接交给handler处理

 <details>
 <summary>
    void HandleNewConnection(experimental::ExternalConnectionAcceptor::NewConnectionParameters* p);
</summary>

```
void ExternalConnectionAcceptorImpl::HandleNewConnection(
    experimental::ExternalConnectionAcceptor::NewConnectionParameters* p) {
  grpc_core::MutexLock lock(&mu_);
  if (shutdown_ || !started_) {
    // TODO(yangg) clean up.
    gpr_log(
        GPR_ERROR,
        "NOT handling external connection with fd %d, started %d, shutdown %d",
        p->fd, started_, shutdown_);
    return;
  }
  if (handler_) {
    handler_->Handle(p->listener_fd, p->fd, p->read_buffer.c_buffer());
  }
}
```

 </details>

