---
title: Linux Kernel Networking Learning -- accept系统调用流程
layout: post
tags:
  - linux
  - kernel
  - network
  - tcp
---

1. 调用sockfd_lookup_light通过fd取struct socket*  

2. 调用sock_alloc创建一个新的struct socket*  

3. 调用sock_alloc_file为struct socket*创建对应的fd  

4. 调用sock->ops->accept（此处为监听套接字！！），实际是调用到inet_stream_ops->accept（inet_accept）  
inet_accept会同时传入监听套接字、连接套接字！  
通过取struct socket* -> sk获取监听套接字对应的struct sock*  
然后调用监听套接字的struct sock*->sk_prot->accept创建连接套接字的struct sock*（sk_prot为tcp_prot，即是inet_csk_accept方法）  
inet_csk_accept方法流程  
  传入的参数为监听套接字的struct sock结构  
  1. 首先判断sock的状态是否为TCP_LISTEN  
  2. 如果icsk->icsk_accept_queue为空，则根据监听套接字是否为nonblock来决定是否等待（sock_rcvtimeo方法获取rcvtimeo等待时间！）（TODO：既然都来了accept了，怎么queue还会是空呢？）  
  3. 有请求连接到来，调用reqsk_queue_get_child出队（返回的是连接套接字的struct sock*），并且将监听套接字的sk->sk_ack_backlog--（TODO：这个队列何时入队的？）  
  4. 返回连接套接字的struct sock*，调用完成  
如果内核定义了CONFIG_RPS，则调用sock_rps_record_flow  
调用sock_graft将struct sock*和struct socket*相映射  
并将struct socket*的状态设置为SS_CONNECTED；  

5. uppeer_sockaddr通过调用newsock->op->getname（这个就是UNP说到的值结果参数）  

6. 调用fd_install将newsock和fd映射起来！
