---
title: Linux Kernel如何new一个sk_buff的  
layout: post
tags:
  - linux
  - kernel
  - network
  - tcp
---

创建sk_buff时候如何决定长度？  
	
	alloc_skb_fclone(size + sk->sk_prot->max_header, gfp)

其中size为  
sk->sk_prot->max_header指向tcp_prot的max_header:  

	#define MAX_TCP_HEADER     (128 + MAX_HEADER)  

而MAX_HEADER在netdevice.h中为：#define MAX_HEADER LL_MAX_HEADER或者#define MAX_HEADER LL_MAX_HEADER + 48，其中LL_MAX_HEADER是根据整个协议栈的内核CONFIG来算出来的最长长度！

