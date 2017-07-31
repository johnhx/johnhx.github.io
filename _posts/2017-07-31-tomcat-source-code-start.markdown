---
title: apache tomcat 8源码学习(1) - start流程
layout: post
categories: Java
tags: Java Tomcat 源码
author: John He
---

* content
{:toc}


## 背景

在改进一个基于一个Servlet Web框架(Draco)的时候, 对Servlet标准产生了些许兴趣(现在已经出到Servlet 4, 支持HTTP/2.0), 接下来便想看一下Servlet的实现。

便翻出了apache tomcat 8的源码, 这一看, 就有点停不下来, 于是从tomcat instance的start流程入手, 对tomcat从上层到底层的源码和机制都梳理一下.

## tomcat的start

tomcat下载后, 我们启动都是通过bin\startup.sh来启动.

startup.sh是一个shell脚本, 调用到的是bin\catalina.sh start, 后跟其他参数.

catalina.sh也是一个shell脚本, 这个脚本稍微复杂了一些, 不过主要都是在处理start, stop, run, debug以及其他参数, 对于start流程, 其核心是调用到下面的Java命令:

```Shell
$JRE_HOME/bin/java ... org.apache.catalina.startup.Bootstrap ... start
```

可见, 对于start的过程, 其入口类是org.apache.catalina.startup.Bootstrap类

## org.apache.catalina.startup.Bootstrap

### 静态区域

主要做了以下几件事情:

- 获取catalinaHomeFile ( 可通过Globals.CATALINA_HOME_PROP获取，默认为bootstrap.jar )

- 获取catalinaBaseFile ( 可通过Globals.CATALINA_BASE_PROP获取 )

### start方法

#### init()

* initClassLoaders()
 + 创建commonLoader
 + 创建catalinaLoader
 + 创建sharedLoader
 
   注意: commonLoader是catalinaLoader和sharedLoader的父Loader.

>  以上创建loader都是通过调用createClassLoader方法:
>
>  1. 获取name + “.loader"配置属性, 对于commonLoader, 就是common.loader
>
>  2. 根据*.loader生成repository列表, 稍后找ClassLoader需要用到
>
>  3. 调用ClassLoaderFactory.createClassLoader方法
>
>  4. 生成java.net.URLClassLoader对象

 + 将catalinaLoader设为currentThread的contextClassLoader

 + 加载org.apache.catalina.startup.Catalina类, 并newInstance

 + 调用org.apache.catalina.startup.Catalina类的setParentClassLoader方法, 将其parent ClassLoader设置为sharedLoader

 + catalinaDaemon赋值为org.apache.catalina.startup.Catalina的instance

* 通过反射调用catalinaDaemon的start方法 (org.apache.catalina.startup.Catalina类实例的startup方法 ):

    - org.apache.catalina.startup.Catalina类startup方法流程
        
        - load()
        
        - Server的start()方法
        
        - 如果await设为true, 则调用await(), stop()


