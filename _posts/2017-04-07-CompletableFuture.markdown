---
title: 一个多线程issue的处理解决过程
layout: post
tags:
  - 多线程
  - Java
---

项目里有一个处理Job的工作流。

实体对应的Job有两个状态：一是实体的Job的状态，以下称为State，另一个是每个实体Job状态的内部状态，以下称为JobState。

其中，State的变迁为业务相关（DOWNLOADRUN, DOWNLOADCHECK, PROCESSRUN, PROCESSCHECK等），JobState为业务不相关（NEW -> INPROGRESS -> COMPLETED）。

主线程负责State的变迁。

用一个线程池Executors.newSingleThreadExecutor来实际处理Job，即负责JobState的变迁。

当线程池中的线程将JobState变为COMPLETED之后，主线程将State前挪，并将JobState重置为NEW，重新让线程池中的线程处理。

伪代码如下：

> listCompletedEntities

> foreach listCompletedEntities


