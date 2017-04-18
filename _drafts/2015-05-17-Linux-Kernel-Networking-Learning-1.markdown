---
title: Linux Kernel Networking Learning -- socket系统调用流程
layout: post
tags:
  - linux
  - kernel
  - network
  - tcp
---

socket接口如下:  

```C
client_sockfd = socket(PF_INET, SOCK_STREAM, 0)  
```

其中，family为PF_INET, type为SOCK_STREAM，protocol为0  

## SOCK层  

主要操作为:
- 调用sock_create创建socket对象；  
- 调用__sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0):  

__sock_create的主要操作为:
- sock_alloc;  
- pf->create(net, sock, protocol, kern), 该方法调到inet_create进行INET层socket的创建;  

## INET层

在net\ipv4\af_inet.c的inet_create中:  

- 遍历inetsw[sock->type]数组(即inetsw[SOCK_STREAM], inetsw_array按照type注册在inetsw中), TCP对应的inetsw结构如下:  
	
```C
...
{
	.type =       SOCK_STREAM,
    .protocol =   IPPROTO_TCP,
    .prot =       &tcp_prot,
    .ops =        &inet_stream_ops,
    .no_check =   0,
    .flags =      INET_PROTOSW_PERMANENT | INET_PROTOSW_ICSK,
}
...
```

- 调用sk_alloc分配sock对象sk
注意：这个sock对象是inet_sock, inet_connection_sock，tcp_sock等通用的sock对象！  

```C	
kmalloc(prot->obj_size, priority);   
```

其中prot->obj_size为INET层中TCP对应的inetsw结构中的&tcp_prot.obj_size, 即sizeof(struct tcp_sock);  
	
sk->sk_family为PF_INET;  
sk->sk_prot为上段代码中的tcp_prot;  
sk->classid为current task的class id;    
sk->sk_cgrp_prioidx和current task的css->cgroup->id保持一致;  
 
- 为inet_sock对象进行初始化  
将sock对象强转为inet_sock进行以下赋值:  
		
inet->is_icsk (如果&tcp_prot的flags包含ICSK，则为1，表示这个sock是一个inet_connection_sock，即面向连接的sock)  
inet->uc_ttl (unicast的ttl)  
inet->mc_ttl  
 
- 调用sock_init_data  
struct sock*为内核实际收发使用的sock  
struct socket*为用户态使用的socket;  
sock_init_data将struct sock*->sk_socket直接赋值为struct socket*;  
初始化sock对象的sk_receive_queue, sk_write_queue, sk_error_queue, 如果有DMA的话，sk_async_wait_queue;  
为以下函数指针挂上callback:  

sk_state_change = sock_def_wakeup  
sk_data_ready = sock_def_readable  
sk_write_space = sock_def_write_space  
sk_error_report = sock_def_error_report  
sk_destruct =inet_sock_destruct

- 调用sk->sk_prot->init(sk)，指向的是tcp_prot中的init即tcp_v4_init_sock  
   *  将sock强转为inet_connection_sock进行以下赋值:  
   ```C
   icsk->icsk_af_ops = &ipv4_specific;  
   ```

   *  调用tcp_init_sock对tcp_sock对象成员进行赋值:  
   out_of_order_queue(专门存放out of order的segment的queue);  
   &icsk->icsk_retransmit_timer挂上tcp_write_timer函数指针;  
   &icsk->icsk_delack_timer挂上tcp_delack_timer函数指针;  
   &sk->sk_timer挂上tcp_keepalive_timer函数指针;
   ucopy结构体（用于direct copy to user）;  
   icsk->icsk_rto初始值为TCP_TIMEOUT_INIT为1*HZ;  
   tcp_sock.mdev（medium deviation);  
   tcp_sock.snd_cwnd（发送的congestion window）;  
   tcp_sock.mss_cache（将max segment size设为536，见self learning.网络学习）;  
   tcp_sock.icsk_ca_ops（congestion control的hook functions）;  
   sk->sk_write_space = sk_stream_write_space（回调函数，根据注释，当bf sending space available时调用），并设置标志位YES to call sk->sk_write_space in sock_wfree;
   sk->sk_sndbuf, sk_rcvbuf（send buffer的大小，receive buffer的大小） = sysctl_tcp_wmem[1] ( need follow-up );
   初始化sk_rcvbuf为全局sysctl_rmem_default, sk_sndbuf为全局sysctl_wmem_default ( need follow-up )


## 05/15/2015总结：

在整个TCP/IP内核协议栈中，原型为struct sock*

在inet_create中，malloc为tcp_sock

然后强转为struct inet_sock*

在TCP做初始化的时候，强转为tcp_sock设置属性，初始化时sk->sk_state为TCP_CLOSE

至此，inet_create流程完全结束!

回到__sock_create的pf->create调用，至此__sock_create基本结束!  
然后调用sock_map_fd来将struct socket*对象和返回给user space的fd进行映射:  
* 调用sock_alloc_file获取struct file*对象和fd;  
* 调用fd_install把struct file*对象和fd加到current进程的fd中去！
