---
title: Linux Socket系统调用内核源码分析
layout: post
categories: Linux
tags: Linux Kernel Network Socket
author: John He
---

* content
{:toc}

这部分代码看了有几年了，归纳总结的想法也想了几年，刚好最近比较空，把这部分代码流程梳理了一下，总结成一篇blog，记录一下。

## 内核初始化相关的流程
内核初始化过程中，依次调到arch/<arch>/boot -> arch/<arch>/kernel -> start_kernel

start_kernel最后调到do_basic_setup()(init/main.c):
```c
static void __init do_basic_setup(void)
{
    cpuset_init_smp();
    usermodehelper_init();
    shmem_init();
    driver_init();
    init_irq_proc();
    do_ctors();
    usermodehelper_enable();
    do_initcalls();
}
```

do_initcalls()(init/main.c)方法如下:
```c
static void __init do_initcalls(void)
{
    initcall_t *fn;

    for (fn = __early_initcall_end; fn < __initcall_end; fn++)
        do_one_initcall(*fn);
}
```

其中__early_initcall_end, __initcall_end定义在Vmlinux.lds.h中:
```c
#define INITCALLS                           \
    *(.initcallearly.init)                      \
    VMLINUX_SYMBOL(__early_initcall_end) = .;           \
    *(.initcall0.init)                      \
    *(.initcall0s.init)                     \
    *(.initcall1.init)                      \
    *(.initcall1s.init)                     \
    *(.initcall2.init)                      \
    *(.initcall2s.init)                     \
    *(.initcall3.init)                      \
    *(.initcall3s.init)                     \
    *(.initcall4.init)                      \
    *(.initcall4s.init)                     \
    *(.initcall5.init)                      \
    *(.initcall5s.init)                     \
    *(.initcallrootfs.init)                     \
    *(.initcall6.init)                      \
    *(.initcall6s.init)                     \
    *(.initcall7.init)                      \
    *(.initcall7s.init)

#define INIT_CALLS                          \
        VMLINUX_SYMBOL(__initcall_start) = .;           \
        INITCALLS                       \
        VMLINUX_SYMBOL(__initcall_end) = .;
```

可见, 在__early_initcall_end和__initcall_end之间是从.initcall0.init到.initcall7s.init这几个段的函数指针.

这几个段里分别放的哪些函数指针呢?

在inlude\linux\init.h中, 有如下的宏定义:
```c
#define pure_initcall(fn)       __define_initcall("0",fn,0)

#define core_initcall(fn)       __define_initcall("1",fn,1)
#define core_initcall_sync(fn)      __define_initcall("1s",fn,1s)
#define postcore_initcall(fn)       __define_initcall("2",fn,2)
#define postcore_initcall_sync(fn)  __define_initcall("2s",fn,2s)
#define arch_initcall(fn)       __define_initcall("3",fn,3)
#define arch_initcall_sync(fn)      __define_initcall("3s",fn,3s)
#define subsys_initcall(fn)     __define_initcall("4",fn,4)
#define subsys_initcall_sync(fn)    __define_initcall("4s",fn,4s)
#define fs_initcall(fn)         __define_initcall("5",fn,5)
#define fs_initcall_sync(fn)        __define_initcall("5s",fn,5s)
#define rootfs_initcall(fn)     __define_initcall("rootfs",fn,rootfs)
#define device_initcall(fn)     __define_initcall("6",fn,6)
#define device_initcall_sync(fn)    __define_initcall("6s",fn,6s)
#define late_initcall(fn)       __define_initcall("7",fn,7)
#define late_initcall_sync(fn)      __define_initcall("7s",fn,7s)
```

可见, 在内核源码中，我们经常看到的诸如core_initcall, arch_initcall, subsys_initcall, fs_initcall宏调用，实际是将函数放到0 ~ 7s对应的段中, 以便于内核在初始化时候进行调用。

比如在内核的网络子系统部分(net/), 网络子系统sysctl interface的初始化逻辑放在net/sysctl_net.c中, 便调用了subsys_initcall宏:
```c
static __init int sysctl_init(void)
{
    int ret;
    ret = register_pernet_subsys(&sysctl_pernet_ops);
    if (ret)
        goto out;
    register_sysctl_root(&net_sysctl_root);
    setup_sysctl_set(&net_sysctl_ro_root.default_set, NULL, NULL);
    register_sysctl_root(&net_sysctl_ro_root);
out:
    return ret;
}
subsys_initcall(sysctl_init);
```


## Socket系统调用流程
Socket系统调用传入的参数为:
```c
int family
int type
int protocol
```

Socket系统调用定义在内核的网络子系统顶层代码中(net\socket.c).

### 一、首先调用到sock_create
```c
sock_create(family, type, protocol, &sock)
```
在这个第一层函数sock_create里, 比起socket系统调用来多出来第四个参数&sock, 

这个参数是一个结构体, 定义如下:
```c
struct socket {
    socket_state        state;
    kmemcheck_bitfield_begin(type);
    short           type;
    kmemcheck_bitfield_end(type);
    unsigned long       flags;
    struct socket_wq __rcu  *wq;
    struct file     *file;
    struct sock     *sk;
    const struct proto_ops  *ops;
};
```

其中:
- state为枚举类型: SS_FREE, SS_UNCONNECTED, SS_CONNECTING, SS_CONNECTED, SS_DISCONNECTED. ( 比较好理解 );
- type为诸如SOCK_STREAM之类的;
- flags是标记;
- wq是一个wait queue, 具体怎么用后面会提到;
- file是供gc使用的文件描述符;
- sk是比较重要的数据结构, 指向代表下层协议(network layer)数据的sock结构

#### 1. __sock_create
sock_create向下调到第二层方法: 
```c
__sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
```
这一层函数引入了一个新的参数struct net, 后面再做分析。

在__sock_create这一层:

##### 1.1 调用secutiry子系统的方法secutiry_ops->socket_create去security子系统折腾了一圈;

##### 1.2 sock_alloc: 分配struct socket
###### 1.2.1 在sockfs中分配一个inode ( sockfs的初始化详见net/socket.c的sock_init方法 )
###### 1.2.2 基于inode结构取到socket结构 ( 怎么取到的? 这里使用的是著名的container_of宏 )

> container_of宏实现如下:
> ```c
> #define container_of(ptr, type, member) ({          \
>    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
>    (type *)( (char *)__mptr - offsetof(type,member) );})
>```
>先new一个指针__mptr，类型同type的member, 指向源数据ptr;
>
>该指针__mptr便存放着源数据ptr的内存地址, 并且有正确的类型;
>
>然后用这个内存地址减去member在type内部的偏移量, 便是包含ptr这个类型为type的结构体的起始地址, 强转为type指针类型, 即为结果.

通过调用container_of宏, 使用inode结构取到了socket结构, 返回.

##### 1.3 从全局net_families数组中根据下标family取到对应的struct net_proto_family结构pf;

全局net_families数组通过sock_register/sock_unregister添加/删除元素. ( 最多40个元素, 可以理解为下层协议的数据结构, 比如ipv4, ipv6, netlink, appletalk, bluetooth等 )

sock_register/sock_unregister通常在net/xxx/Af_xxx.c中调用.

例如对于INET, 在net/ipv4/Af_inet.c中将INET的struct net_proto_family结构添加到全局net_families数组中. 

INET的struct net_proto_family定义如下:
```c
static const struct net_proto_family inet_family_ops = {
    .family = PF_INET,
    .create = inet_create,
    .owner  = THIS_MODULE,
};
```

##### 1.4 关键的一步调用到pf->create, 按照上面的描述, 调用到了net/ipv4/Af_inet.c的inet_create方法.

1. 先将struct socket的state设为SS_UNCONNECTED;

2. 根据struct socket的type(SOCK_STREAM之类), 遍历inetsw[type], 找到对应到protocol的结构体;

注意: 

inetsw是一个链表数组, key为SOCK_STREAM, SOCK_DGRAM, SOCK_RAW等等.

inetsw的初始化在net/ipv4/Af_inet.c的inet_init方法中:

- 先初始化inesw为SOCK_STREAM/SOCK_DGRAM/SOCK_RAW等作为key的list_head数组;
- 遍历inetsw_array, 将其挂入inetsw中. 

inetsw_array封装了TCP, UDP, ICMP等协议的处理逻辑, 

为上文中描述的"对应到protocol的结构体", inetsw_array的元素结构如下:
```c
static struct inet_protosw inetsw_array[] =
{
    {
        .type =       SOCK_STREAM,
        .protocol =   IPPROTO_TCP,
        .prot =       &tcp_prot,
        .ops =        &inet_stream_ops,
        .no_check =   0,
        .flags =      INET_PROTOSW_PERMANENT |
                  INET_PROTOSW_ICSK,
    }
...
}
```

3. 将"对应到protocol的结构体"的ops赋给struct socket结构的ops. 

例如如果type是SOCK_STREAM, protocol是TCP, 将&inet_stream_ops赋给struct socket结构的ops.

4. 调用sk_alloc, 分配网络子系统核心(net/core)的数据结构struct sock ( 记录family, protocol到sk_family, sk_prot成员 )

struct sock的sk_prot指向“对应到protocol的结构体”中的prot, 即&tcp_prot;

5. 调用inet_sk, 分配inet层的数据结构struct inet_sock

注意:

如果是SOCK_RAW, 会把protocol信息直接存放在struct inet_sock的inet_num成员 ( local port? )

6. 根据ipv4_config.no_pmtu_disc, 设置struct inet_sock的pmtudisc标志位DONT或者WANT.

注意:

PMTUD全称Path MTU Discovery, 指的是在两个IP host间决定Maximum Transmission Unit ( MTU )的技术, 目的是避免IP分片.

由此可见, 这个PMTUD在内核里面是可配的!!!

7. 调用sock_init_data(struct socket, struct sock)

从函数原型上可以看出, 是借助struct socket来初始化网络层核心数据结构struct sock:

- sk->sk_receive_queue, sk_write_queue, sk_error_queue ( 如果配了NET_DMA, 还有sk_async_wait_queue )
- sk->sk_send_head
- sk->sk_timer
- sk->sk_rcvbuf ( 这里指的是rmem )
- sk->sk_sndbuf ( 这里指的是wmem )
- sk->sk_state ( 初始值为TCP_CLOSE )
- 接管struct socket的wq
- sk->sk_dst_lock
- sk->sk_callback_lock
- sk->sk_rcvlowat ( SO_RCVLOWAT )
- sk->sk_rcvtimeo ( SO_RCVTIMEO )
- sk->sk_sndtimeo ( SO_SNDTIMEO )
- sk->sk_stamp ( 上一个packet收到的timestamp )

8. sk->sk_backlog_rcv ( 收到backlog的回调函数 ) 初始化为sk->sk_prot->backlog_rcv

例如对于TCP, backlog_rcv指向net/ipv4/Tcp_ipv4.c的全局结构体struct proto tcp_prot中的tcp_v4_do_rcv

9. inet->ut_ttl ( Unicast的TTL )为1

10. inet->mc_loop ( Multicast的loop ) 为1

11. inet->mc_ttl ( Multicast的TTL ), mc_all为1

12. inet->mc_index ( Multicast的device index )为0

13. 如果inet->inet_num ( local port? )不为空, 

意味着该protocol允许并且已经在socket创建时指定local Port, 于是调用sk->sk_prot->hash(sk).

例如对于TCP, hash()指向net/ipv4/Tcp_ipv4.c的全局结构体struct proto tcp_prot中的inet_hash:

该方法按local Port计算hash key, 在&tcp_hashinfo->listening_hash按hash key, 将struct sock插入tcp的listening_hash链表.

但是似乎这个hashtable元素只有32个, 为什么这么小? (待看)

14. 调用sk->sk_prot->init, 例如对于TCP, 指向net/ipv4/Tcp_ipv4.c的全局结构体struct proto tcp_port中的tcp_v4_init_sock, 此方法完成该socket在内核网络子系统TCP层的初始化:

注意: TCP层有自己的数据结构struct tcp_sock, 从struct sock强转而来

- out_of_order_queue初始化 ( 该queue用于存放Out of order的segment )
- 初始化三个timer: 
retransmit_timer <-> net/ipv4/tcp_timer.c的tcp_write_timer
delack <-> net/ipv4/tcp_timer.c的tcp_delack_timer
keepalive <-> net/ipv4/tcp_timer.c的tcp_keepalive_timer
- "Data for direct copy to user" ( 称为prequeue )的init ( 机制待细看 )
- RTO ( Retransmit Time Out ) 初始化为1*HZ
- mdev 初始化为1*HZ ( mdev是什么待细看 )
- snd_cwnd初始化为10 ( Sending Congestion Window )
- snd_ssthresh初始化为0x7fffffff ( 慢启动size threshold )
- snd_cwnd_clamp初始化为~0 ( snd_cwnd的上限 )
- mss_cache初始化为536U
- reordering初始化为sysctl_tcp_reordering ( packet reording metric )
- icsk->icsk_ca_ops ( pluggable congestion control hook 可插拔的拥塞控制hook )
- sk->sk_state设为TCP_CLOSE
...

至此pf->create调用结束, 也就是inet_create方法调用结束.

##### 调用secutiry子系统的方法secutiry_ops->socket_post_create去security子系统折腾了一圈;

至此__sock_create调用结束.

至此socket系统调用中sock_create调用结束.

### 然后调用sock_map_fd将struct socket挂接到fd供进程使用.

至此, 整个socket系统调用结束.