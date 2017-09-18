---
title: Https的SSL/TLS协议分析
layout: post
categories: Https
tags: Networking Https
author: John He
---

* content
{:toc}

## 背景

关于Https协议规范, "Https协议详解"一文作为入门描述很不错.

本文基于访问https://mail.google.com的WireShark抓包结果分析Https协议过程, 操作过程为:

1. 打开chrome浏览器并访问https://mail.google.com首页

2. 关闭chrome

这个过程客户端chrome和mail.google.com之间新建了3个TCP连接.

为便于描述, 通过Wireshark筛选了其中一个连接的数据做分析.

总体上讲，经历了一下几个大的阶段:

1. TCP的三路握手

2. TLSv1.2协议握手

3. 应用数据传输

4. 断连

## 第一阶段 TCP的三路握手

没什么好说的，见图:


![Image of TCP Handshake](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/https/tcp_handshake.png)


## 第二阶段 TLSv1.2协议握手

- 第一步 客户端往服务端443端口发一个Client Hello报文

应用层协议为Secure Socket Layer, 如下图所示:


  ![Image of Client Hello](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/https/client_hello.png)


Secure Socket Layer的具体内容如下图所示, 包括：

  - Content Type为Handshake

  - Version为TLS v1.0

  - Length表示Secure Socket Layer的数据长度
  
  - Handshake Protocol的详细内容:
  
    - Handshake Type为Client Hello, 值为1

    - Length表示Handshare Protocol的数据长度
 
    - Version为TLS 1.2

    - 随机数: 这是客户端生成的随机数, 稍后用于生成"对话密钥"

    - Session ID Length为0(还没有Session ID生成, 故Length为0)

    - Cipher Suites: 客户端支持的加密方法

    - Compression Methods: 客户端支持的压缩方法

    - Extension部分(略)


  ![Image of Client Hello SSL](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/https/client_hello_ssl.png)


- 第二步 服务端首次回应Server Hello报文

  应用层协议同样为Secure Sockets Layer, 具体内容如下图所示:

  - Content Type为Handshake

  - Version为TLS v1.2

  - Length表示Secure Socket Layer的数据长度

  - Handshake Protocol的详细内容:

    - Handshake Type为Server Hello, 值为2

    - Length表示Handshare Protocol的数据长度

    - Version为TLS 1.2

    - 随机数: 这是服务端生成的随机数 ( 这是Https协议握手过程的第二个随机数 )

    - Session ID Length为0(还没有Session ID生成, 故Length为0)

    - Cipher Suite: 服务端决定的加密算法

    - Compression Methods: 服务端决定的压缩方法, 此处为null

    - Extension部分

  需要注意的是, 在协议的Extension部分, 服务端将后面步骤用到的证书的时间戳放在signed_certificate_timestamp字段发给了客户端.


  ![Image of Server Hello](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/https/server_hello.png)


- 第三步 服务端将证书发给客户端

  这个报文比较长, 是3个TCP分节的combine.

  逻辑上包含三个TLSv1.2 Record Layer的报文:

  - Handshake Protocol为"Certificate(11)": 包含证书
  
  - Handshake Protocol为"Server Key Exchange(12)": 包含Diffie-Hellman算法的服务端证书公钥, 证书本身的数字签名

  - Handshake Protocol位"Server Hello Done(14)": 表示服务端的Server Hello完成

  
  ![Image of Certificates](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/https/certificates.png)


- 第四步 客户端回应服务端

  
  逻辑上包含如下几个TLSv1.2 Record Layer的报文:

  - Handshake Protocol为"Client Key Exchange(16)": 包括Diffie-Hellman算法的客户端公钥

  - Change Cipher Spec报文: 告知服务端, 客户端已经切换到之前的Cipher Suite来加密数据并传输

  - 第三个随机数的加密数据

  此外, 客户端使用前面的两个随机数, 已经刚刚生成的第三个随机数, 使用之前与服务器确定的加密算法, 在客户端生成一个Session Secret.


  ![Image of Client Key Exchange](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/https/client_key_exchange.png)


- 第五步 服务端再次响应客户端

  逻辑上包含如下几个TLSv1.2 Record Layer报文:

  - Handshake Protocol为"New Session Ticket(4)"的报文: 包含Session Ticket
  
  - Change Cipher Spec报文: 告知客户端, 服务端已经切换到之前的Cipher Suite来加密数据并传输
  
  - 加密的Finish消息 ( 用于验证之前通过握手建立起来的加解密通道是否成功 )


  ![Image of new session ticket](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/https/new_session_ticket.png)


至此, TLSv1.2的握手过程完成。

往后, 服务端和客户端就可以通过TLSv1.2传输加密数据了, 加密数据的TLSv1.2 Record Layer的Content Type为Application Data(23).


## 总结

SSL/TLS在握手阶段使用的是非对称加密, 在传输阶段使用的是对称加密.

传输阶段的对称加密的密钥是基于非对称算法在不安全的网络上让会话双方在握手阶段生成的。

兼顾了安全和性能。
