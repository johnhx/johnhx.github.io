---
title: Linux内核网络子系统源码分析(2) -- connect系统调用
layout: post
categories: Linux内核
tags: Linux内核 Network Socket
author: John He
---

* content
{:toc}

接[《Linux内核网络子系统源码分析(1) -- socket系统调用和关键数据结构》]("http://johnhx.github.io/2017/05/01/Socket-flow/")一文, 继续往下, 阅读Linux内核子系统connect系统调用流程源码.

由于网络子系统数据结构交错复杂, 先上一张数据结构关系图, 便于后文描述:

![Image of struct socket initialization](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/socket_struct_init.png)

## connect系统调用
connect系统调用的作用为连接server端正在listen的socket, connect系统调用由client端调用, 接口原型为:
```c
int connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);
```

connect系统调用实现入口位于Linux内核的网络子系统顶层代码中(net\socket.c).

暴露出来的connect接口指向函数为kernel_connect:
```c
int kernel_connect(struct socket *sock, struct sockaddr *addr, int addrlen,
           int flags)
{
    return sock->ops->connect(sock, addr, addrlen, flags);
}
EXPORT_SYMBOL(kernel_connect);
```

对于net\ipv4\这个family来说, 结合文首图得知, sock->ops指向inetsw_array对应type的ops.

type为SOCK_STREAM, prot指向tcp_prot, ops指向inet_stream_ops

sock->ops->connect指向inet_stream_ops的connect, 即inet_stream_connect.

### inet_stream_connect流程

#### 1. 取struct socket内部的struct sock成员指针指向的struct sock; (见文首图很清晰)

#### 2. 根据struct socket的socket_state类型字段:

- SS_CONNECTED/SS_CONNECTING(略过)

- SS_UNCONNECTED: 调用struct sock的sk_prot的connect函数( 这里即指向tcp_proc的connect, 方法名为tcp_v4_connect )

### tcp_v4_connect流程

#### 1. 从connect系统调用传入的struct sockaddr取出destination address, destination port, 并取得nexthop(如果struct inet_sock的struct ip_options_rcu有设置, nexthop为struct ip_option的faddr), 然后调用ip_route_connect进行寻路, 取得struct rtable;

问题: struct inet_sock的struct ip_options_rcu *inet_opt在何处设置?

#### 2. 根据IP寻路的结果, 确定destination address, destination port

#### 3. 设置默认TCP_MSS  ( IPv4默认为536 )

#### 4. tcp状态设为TCP_SYN_SENT
```c
tcp_set_state(sk, TCP_SYN_SENT);
```

#### 5. 将struct sock插入bind链表中
```c
err = inet_hash_connect(&tcp_death_row, sk);
```

*在这一步之前source port可能还为0(对于client端调用socket后直接调用connect不经过bind的情况, source port为0).

所以inet_hash_connect方法会生成一个随机的source port, 赋给struct inet_sk的inet_sport成员.*

#### 6. 调用ip_route_newports;

#### 7. 为tcp生成sequence number, 赋给struct sock的write_seq成员;

#### 8. 调用tcp_connect方法

##### 8.1 分配struct sk_buff

##### 8.2 在struct sk_buff中的tcp_skb_cb部分的tcp_flags字段, 设置SYN标志, 设置seq, end_seq, gso_* (什么是gso?)

##### 8.3 根据sysctl_tcp_ecn, 设置tcp_flags字段中的ECN, CWR标志

##### 8.4 给struct sk_buff打上时间戳

##### 8.5 减struct sk_buff引用

##### 8.6 调用__skb_queue_tail, 将struct sk_buff挂到struct sock的sk_write_queue

##### 8.7 调用tcp_transmit_skb方法进入**tcp层实际的处理和传输过程**

### tcp_transmit_skb流程

**此方法开始TCP层实际的头部数据处理和往下层传输.**

tcp_transmit_skb方法接口如下:

```c
static int tcp_transmit_skb(struct sock *sk, struct sk_buff *skb, int clone_it, gfp_t gfp_mask)
```

#### 1. 如果clone_it为1，调用skb_clone方法clone一个sk_buff, 注意此处只是克隆数据结构，packet data只是加引用计数;

skb_clone方法克隆的struct sk_buff不属于任何socket.

#### 2. 计算tcp头部空间大小tcp_header_size: sizeof(struct tcphdr) + tcp_options_size

#### 3. 使用tcp_header_size来构建tcp头部, 包括设置source port, dest port, seq, ack_seq, window等

#### 4. 调用struct inet_connection_sock的icsk_af_ops的queue_xmit方法 ( 该函数指针在tcp_v4_init_sock方法中初始化, 指向ip_output.c的ip_queue_xmit )

### ip_queue_xmit方法 (IP层实际的处理和传输过程)

#### 1. 加ip头部

#### 2. ip_local_out -> __ip_local_out -> nf_hook(IPV4, LOCAL_OUT, ...), 此处进入netfilter子系统, 若允许pass, 返回1, 调用dst_output

此处是第一次碰到netfilter子系统, 在Linux内核网络子系统中, Netfilter相当于一个hook系统, 在不同的时间点进行filter, 有如下几个大的时间点:

```c
enum nf_inet_hooks{
    NF_INET_PRE_ROUTING,
    NF_INET_LOCAL_IN,
    NF_INET_FORWARD,
    NF_INET_LOCAL_OUT,
    NF_INET_POST_ROUTING,
    NF_INET_NUMHOOK
}
```

#### 3. dst_output方法:
```c
return skb_dst(skb)->output(skb);
```

struct dst_entry的output方法函数指针在net\ipv4\route.c的__mkroute_output得到赋值, 指向ip_output方法.

### ip_output方法

#### 1. 获取struct net_device

#### 2. 再次调用netfilter, 进行POST_ROUTING的filter. filter的结果为pass则调用ip_finish_output.

ip_finish_output -> 是否需要分片? ip_fragment() : ip_finish_output2()

先描述ip_finish_output2()这条线的流程.

#### 3. 获取struct dst的struct neighbour成员

struct neighbour有一个NUD信息, 全称为“neighbour unreachability detection”;
(此处应该补一张structure neighbour的结构图)

依次调用neigh_output -> neigh_hh_output, 最终调用到dev_queue_xmit方法

### dev_queue_xmit方法

struct net_device -> struct netdev_queue -> struct Qdisc

调用struct Qdisc的enqueue方法将struct sk_buff挂入队列中.

struct Qdisc有多种实现, 实现在net\sched\sch_*.c中, 有fifo, generic, dsmark, cbq, choke等(留待日后细分析).

一个connect过程的调用框图如下:

![Image of connect sequence](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/connect_sequence.png)
