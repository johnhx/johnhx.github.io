---
title: 记录一个的Spring Boot Maven Plugin的issue
layout: post
categories: Java
tags: Java Spring Maven
author: John He
---

* content
{:toc}

## 背景

最近几天在把一个Restful后台项目从Spring MVC移植到Spring Boot.

移植的目的有两个:

一是趁机会学习一下Spring Boot;

二是Spring Boot Secutity提供了很好的Restful API的鉴权接口.

移植的过程遇到不少问题, 尤以本文打算记叙的这个问题最为奇葩.

## 问题介绍

这个Restful后台项目所用的数据库是MS SQL Server, SQL Server的JDBC Driver, 是在微软官网上提供的jar包下载, 似乎并没有任何Maven Repository对此jar包做官方提供.

在这样的背景下, 我能想到的最佳引用方式就是在pom.xml引用本地lib目录的jar包, 如下代码:

```xml
<dependency>
    <groupId>com.microsoft.sqlserver</groupId>
    <artifactId>sqljdbc42</artifactId>
    <version>4.2</version>
    <scrope>system</scrope>
    <systemPath>${project.basedir}/lib/sqljdbc42-4.2jar</systemPath>
</dependency>
```

然后再在spring-boot-maven-plugin的配置中将includeSystemScope设置为true:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        ...
        <includeSystemScope>true</includeSystemScope>
        ...
    </configuration>
</plugin>
```

这样在编译jar包的时候, sqljdbc42-4.2.jar被打在了BOOT-INF\lib里面.

用mvn spring-boot:run可以正确加载并且运行.

但是该项目是一个Web应用, 最终是要发布在tomcat里面的, 所以在发布之前是要被打成war包的.

问题来了, 当在pom.xml里加上如下选项, 将其打为war包时, sqljdbc42-4.2.jar被打进了WEB-INF\lib-provided里面, 而不是期望的WEB-INF\lib.

```xml
<packaging>war</packaging>
```

maven的scope分为如下几种:

- compile
  
  默认Scope. 设置为compile scope的依赖, 在编译, 测试, 运行阶段, 这个artifact对应的包都会在classpath中, 意味着, 设置为compile scope的依赖, 会被打入发布的jar, war包之中.

- provided
  
  provided指的是目标容器已经provide了这个依赖包/artifact. 所以设置为provided scope的依赖, 只在编译, 测试阶段会出现在classpath中.

  而在运行阶段, 由于我们假定目标容器会provide这个依赖包/artifact, 所以这个依赖包不会被打包进入发布的jar, war之中.

- runtime

  运行时scope. 只是在测试, 运行阶段会出现在classpath中.

- test

  测试scope, 只有在测试编译和测试运行阶段可用.

- system

  system与provided类似, 但是必须显式的提供一个本地文件系统中jar文件的路径. 将一个依赖设置为system的同时, 必须提供一个systemPath, 指明本地文件系统中jar文件的路径.


## 问题的分析和解决

从maven的scope的说明文档上, system这个scope是不被推荐使用的.

而且"system和provided类似”, 似乎本问题(sqljdbc42-4.2.jar被打进war包的WEB-INF\lib-provided而不是WEB-INF\lib)的存在也是合理的.

所以解决这个问题无外乎几个办法:

- 引用公共或自己定制的Maven Repository中的sqljdbc42-4.2.jar

- 看spring-boot-maven-plugin是否有提供选项将system scope的jar打包路径从WEB-INF\lib-provided变为WEB-INF\lib

自己定制Maven Repository这个办法太麻烦了.

考虑到发布的便利, 我决定优先采用第二种办法.

下了一份spring-boot的源码(包含spring-boot-maven-plugin的源码), 读了war包打包部分的逻辑, 发现并没有办法动态配置指定scope或特定jar包的打包路径.

maven本身有提供<resources>这样的标签, 但是这个是用于定制target/目标目录结构的.

本文问题是出在war包的打包结构上, 为打war包负责的模块应该是spring-boot-maven-plugin.

居然没有提供这样的动态配置接口...


