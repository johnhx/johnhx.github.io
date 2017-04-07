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

``

while(on){
	var completedEntities = listCompletedEntities

	foreach listCompletedEntities
		update nextState
		set JobState = New
	/foreach

	var startOrNewEntities = listStartOrNewEntities

	foreach startOrNewEntities
		CompletedFuture future = CompletedFuture.supplyAsync(() -> {
			set JobState = INPROGRESS
			// hanlde jobs
		}, jobExecutor)	// jobExecutor为newSingleThreadExecutor所创建
	
		future.thenAccept
			set JobState = COMPLETED
	/foreach
}

``

附上newSingleThreadExecutor的背景知识：

> ThreadExecutorPool的构造函数有多个参数

- corePoolSize: the number of threads to keep in the pool

- maximumPoolSize: the maximum number of threads to allow in the pool

- keepAliveTime: 当线程数量大于corePoolSize时，当一个线程无事可做时，超过一定的时间（keepAliveTime），这个线程就会被停掉。

- unit

- workQueue：类型为BlockingQueue<Runnable>

newSingleThreadExecutor创建的corePoolSize为1，maximumPoolSize为1，workQueue为unbounding的LinkedBlockingQueue，即无界队列，允许无限多排队。


