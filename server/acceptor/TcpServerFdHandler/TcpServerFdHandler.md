# TcpServerFdHandler


## ExternalConnectionHandler

```
class ExternalConnectionHandler : public grpc_core::TcpServerFdHandler {
 public:
  explicit ExternalConnectionHandler(grpc_tcp_server* s) : s_(s) {}

  // TODO(yangg) resolve duplicate code with on_read
  void Handle(int listener_fd, int fd, grpc_byte_buffer* buf) override {
    grpc_pollset* read_notifier_pollset;
    grpc_resolved_address addr;
    memset(&addr, 0, sizeof(addr));
    addr.len = static_cast<socklen_t>(sizeof(struct sockaddr_storage));
    grpc_core::ExecCtx exec_ctx;
    
    //获取socket的对方地址
    if (getpeername(fd, reinterpret_cast<struct sockaddr*>(addr.addr),
                    &(addr.len)) < 0) {
      gpr_log(GPR_ERROR, "Failed getpeername: %s", strerror(errno));
      close(fd);
      return;
    }
    grpc_set_socket_no_sigpipe_if_possible(fd);
    std::string addr_str = grpc_sockaddr_to_uri(&addr);
    if (grpc_tcp_trace.enabled()) {
      gpr_log(GPR_INFO, "SERVER_CONNECT: incoming external connection: %s",
              addr_str.c_str());
    }
    std::string name = absl::StrCat("tcp-server-connection:", addr_str);
    
    // 将fd包装成grpc_fd
    grpc_fd* fdobj = grpc_fd_create(fd, name.c_str(), true);
    read_notifier_pollset =
        (*(s_->pollsets))[static_cast<size_t>(gpr_atm_no_barrier_fetch_add(
                              &s_->next_pollset_to_assign, 1)) %
                          s_->pollsets->size()];
    
    // 添加fd到pollset中
    grpc_pollset_add_fd(read_notifier_pollset, fdobj);
    grpc_tcp_server_acceptor* acceptor =
        static_cast<grpc_tcp_server_acceptor*>(gpr_malloc(sizeof(*acceptor)));
    
    acceptor->from_server = s_;
    acceptor->port_index = -1;
    acceptor->fd_index = -1;
    acceptor->external_connection = true;
    acceptor->listener_fd = listener_fd;
    acceptor->pending_data = buf;
    
    s_->on_accept_cb(s_->on_accept_cb_arg,
                     grpc_tcp_create(fdobj, s_->channel_args, addr_str.c_str()),
                     read_notifier_pollset, acceptor);
  }

 private:
  grpc_tcp_server* s_;
};
```