---
title: Linux Kernel Networking Learning -- listen系统调用流程
layout: post
tags:
  - linux
  - kernel
  - network
  - tcp
---

调用sock->ops->listen，实际是调用到inet_stream_ops->listen指针( inet_listen )  
listen传进的最后一个参数是backlog!!  

inet_listen流程:  
调用inet_csk_listen_start（同样传入struct sock*和backlog):  
开一个backlog大小的queue，赋给icsk->icsk_accept_queue  
将sk->sk_state改为TCP_LISTEN  
似乎listen的操作只是改TCP的状态和申请queue？？？  
