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

```shell
$JRE_HOME/bin/java ... org.apache.catalina.startup.Bootstrap ... start
```

可见, 对于start的过程, 其入口类是org.apache.catalina.startup.Bootstrap类

## org.apache.catalina.startup.Bootstrap

### 静态区域

主要做了以下几件事情:

- 获取catalinaHomeFile ( 可通过Globals.CATALINA_HOME_PROP获取，默认为bootstrap.jar )

- 获取catalinaBaseFile ( 可通过Globals.CATALINA_BASE_PROP获取 )

### start方法

#### <b>init()获取catalinaDaemon</b>

  - initClassLoaders()

    - 创建commonLoader

    - 创建catalinaLoader

    - 创建sharedLoader

    注意: commonLoader是catalinaLoader和sharedLoader的父Loader.

    > 以上创建loader都是通过调用createClassLoader方法, createClassLoader方法流程如下:
    > 1. 获取name + “.loader"配置属性, 例如对于commonLoader, 就是common.loader
    > 2. 根据*.loader生成repository列表, 稍后找ClassLoader需要用到
    > 3. 调用ClassLoaderFactory.createClassLoader方法
    > 4. 生成java.net.URLClassLoader对象

  - 将catalinaLoader设为currentThread的contextClassLoader

  - 加载org.apache.catalina.startup.Catalina类, 并newInstance

  - 调用org.apache.catalina.startup.Catalina类的setParentClassLoader方法, 将其parent ClassLoader设置为sharedLoader

  - catalinaDaemon赋值为org.apache.catalina.startup.Catalina的instance

#### <b>通过反射调用catalinaDaemon的start方法:</b>

org.apache.catalina.startup.Catalina类startup方法流程:

- <b>load()</b>
  
  - initDirs(): 检查系统配置java.io.tmpdir

  - createStarterDigest

    创建org.apache.tomcat.util.digest实例

    注意: Digester是个维护XML和Java对象的库( 见 [Apache Digester](https://commons.apache.org/proper/commons-digester/guide/core.html) ), 动态添加了很多规则, 如:

    - 当XML Parser扫描到“<Server>”时, 创建org.apache.catalina.core.StandardServer实例, 并保持其生命周期到“</Server>”
     
    - 当XML Parser扫描到“<Server><GlobalNamingResources>”时, 创建org.apache.catalina.deploy.NamingResourceImpl实例
     
      ...

      ( 详见Catalina.java\createStarterDigester方法 )

      个人感受: Digester是一个另类的依赖注入模型

  - 取conf/server.xml, 并由digest进行解析

    ```java
    digest.push(this)           // 将Catalina.java类实例作为Digest的object stack的root
    digest.parse(inputSource)   // 解析
    ```

    这样Catalina.java的getServer()返回的便是Digestor的规则所创建的Server实例( org.apache.catalina.core.StandardServer )

  - 调用StandardServer的init

    附: StandardServer的继承关系如下图:


    ![Image of StandardServer](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/tomcat_1_standardserver.png)


    附: Lifecycle的状态机如下:


    ![Image of Lifecycle State Machine](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/tomcat_1_lifecyclestate.png)


    <b> 调用StandardServer的init, 但是因为StandardServer没有init方法, 实际调用到的是LifecycleBase的init: </b>

    <u>LifecycleBase的init()流程:</u>

    - Lifecycle状态迁移为INITIALIZING

    - 调用initInternal(), 调用到的是StandardServer的initInternal:

      - 新建StringCache对象, 注册Catalina:type=StringCache到MBean Server;

      - 新建MBeanFactor对象, 将container设置为当前的StandardServer实例, 注册Catalina:type=MBeanFactory到MBean Server;

      - 调用NamingResourceImpl的initInternal:
      
        - 为resources, env, resourceLinks调用createMBean

      - 把common和shared的classes添加到ExtensionValidator

      - 调用services的init() ( services是由conf\server.xml的<Service>节点添加 )

        <u>StandardService的init()流程:</u>

        - 调用Executor的init 

          注意: Executor是在被Digestor在解析conf\server.xml时, 解析到<Server><Service><Executor>时添加, <Executor>用于定义线程池.

        - 初始化MapperListener实例 ( 用于listen virtualhosts的配置改变 )

        - 调用Connector的init

          注意: Connector是在被Digestor在解析conf\server.xml时, 解析到\<Server\>\<Service\>\<Connector\>时添加. 

          Digester在XML Parser到如下节点时:
        
          ```xml
          ...
          <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
          ...
          ```

          调用ConnectorCreateRule的begin()方法, 创建Connector对象, 放入digester的object stack栈顶.

          <u>创建Connector对象流程:</u>

          - 获取传入的protocol ( "HTTP/1.1", "AJP/1.3" )

          - 加载protocolHandlerClassName类 ( org.apache.coyote.http11.Http11NioProtocol )

          - newInstance创建Http11NioProtocol类实例, 赋值给this.protocolHandler

          <u>/创建Connector对象流程</u>

          <u>Connector的init流程:</u>

          - 创建CoyoteAdapter类实例, 赋值给protocolHandler的Adapter；

          - 初始化protocolHandler ( protocolHandler.init, 即调到Http11NioProtocol的init )

            <u>Http11NioProtocol的init流程:</u>

            Http11NioProtocol的类结构如下:


            ![Image of NioProtocol](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/tomcat_1_nioprotocol.png)


            **Http11NioProtocol类自身没有init方法, 故init方法调到的是父类AbstractHttp11JsseProtocol的init:**

            <u>AbstractHttp11JsseProtocol的init()流程:</u>

            - 获取SSLImplementation类实例

            - AbstractProtocol的init方法:

              <u>AbstractProtocol的init()流程:</u>

              - 生成Catalina:type=ProtocolHandler,port=8080,address=xxx的ObjectName实例, 注册到MBean Server

              - 生成Catalina:type=ThreadPool,name=xxx的ObjectName实例, 注册到MBean Server

              - 生成Catalina:type=GlobalRequestProcessor,name=xxx的ObjectName实例, 注册到MBean Server

              - 调用endpoint的init()方法, 这里endpoint指向NioEndpoint实例:

                <u>NioEndpoin的init()流程</u>

                - 调用bind()方法 ( 调到NioEndpoint的bind方法 ):
                  
                  - java.nio的调用:

                    ServerSocketChannel.open() -> bind, configureBlocking, setSoTimeout

                  - org.apache.tomcat.util.net.NioSelectorPool的open调用

                    - java.nio.channels.Selector的open调用

                    - 新的org.apache.tomcat.util.net.NioBlockingSelector实例

                    - NioBlockingSelector的open调用:

                      - 创建新的BlockPoller线程实例 ( NioBlockingSelector.BlockPoller-线程计数 )

                      - BlockPoller的start调用 ( run方法 ):
                      
                        - while循环

                        - 先处理events( 读, 写事件 )

                        - select

                        - dispatch select出来的事件

                - bindState置为BOUND_ON_INIT;

                <u>/NioEndpoin的init()流程</u>

              <u>/AbstractProtocol的init()流程</u>

            <u>/AbstractHttp11JsseProtocol的init()流程</u>

            <u>/Http11NioProtocol的init流程</u>

          <u>/Connector的init流程</u>

        <u>/StandardService的init()流程</u>

    - Lifecycle状态迁移为INITIALIZED

    <u>/LifecycleBase的init()流程</u>

- <b>Server的start()方法</b>

  **调用StandardServer的start, 但是由于StandardServer没有start方法, 实际调用到的是LifecycleBase的start:**

  <u>LifecycleBase的start()流程:</u>

    - Lifecycle状态迁移为STARTING_PREP

    - 调用startInternal ( 这里调到StandardServer的startInternal方法 )

      <u>StandardServer的startInteral()流程</u>

        - 触发Lifecycle的事件CONFIGURE_START_EVENT

        - Lifecycle状态设为STARTING

        - 调用service的start

          **调用StandardService的start，但是由于StandardService没有start方法, 实际调到LifecycleBase的start, 再调到StandardService的startInternal**



          ![Image of StandardService](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/tomcat_1_standardservice.png)


          <u>StandardService的startInteral()流程</u>

            - Lifecycle状态迁移为STARTING

            - 调用Executor的start()

            - MapperListener实例的start()

            - Connector的start()

              <u>Connector的startInternal()流程 ( 继承自LifecycleBase和Lifecycle )</u>

                - Lifecycle状态迁移为STARTING

                - 调用protocolHanlder对象的start()

                  <b>Http11NioProtocol的start()流程, 实际调到AbstractProtocol的start()流程
                  endpoint的start(), 实际调到NioEndpoint的start()</b>

                  <u>NioEndpoint的startInternal()流程</u>

                    - 创建processCache, 默认大小128, 默认最大500
                    
                    - 创建eventCache, 默认大小128, 默认最大500

                    - 创建nioChannel, 默认大小128, 默认最大500

                    - 如果没有指定Executor, 创建新的Executor, 为new ThreadPoolExecutor, 默认核心线程数为10, 默认最大线程数为500, keepAliveTime为60... ( 此Executor为read/write用 )

                    - initializeConnectionLatch ( 限流用的? ), 默认连接数10000

                    - 创建Runnable的对象Poller, 个数为处理器的个数

                    - 启动Poller ( 每个Poller线程的命名规范为*-ClientPoller-* )

                    - 创建Acceptor的线程 ( Acceptor线程的命名规范为*-Acceptor-* )

                  NioEndpoint从server的角度看来, 在startInternal流程结束后, 线程模型如下图:



                  ![Image of NioEndpoint](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/tomcat_1_nioendpoint.png)


                  <u>/NioEndpoint的startInternal()流程</u>

              <u>/Connector的startInternal()流程 ( 继承自LifecycleBase和Lifecycle )</u>

          <u>/StandardService的startInteral()流程</u>

      <u>/StandardServer的startInteral()流程</u>

    - Lifecycle状态迁移为STARTED

  /LifecycleBase的start()流程</u>


- <b>如果await设为true, 则调用await(), stop()</b>


