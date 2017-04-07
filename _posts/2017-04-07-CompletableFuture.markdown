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

```java

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

```

附上newSingleThreadExecutor的背景知识：

> ThreadExecutorPool的构造函数有多个参数
> - corePoolSize: the number of threads to keep in the pool
> - maximumPoolSize: the maximum number of threads to allow in the pool
> - keepAliveTime: 当线程数量大于corePoolSize时，当一个线程无事可做时，超过一定的时间（keepAliveTime），这个线程就会被停掉。
> - unit
> - workQueue：类型为BlockingQueue<Runnable>

> newSingleThreadExecutor创建的corePoolSize为1，maximumPoolSize为1，workQueue为unbounding的LinkedBlockingQueue，即无界队列，允许无限多排队。

###### issue描述

同时launch多个Job（比如2个），称为Job1，Job2.

Job1和Job2同时进入NEW状态。

Job1被process为COMPLETED，准备进入下一状态；

Job2随后被process为COMPLETED，也准备进入下一状态。

问题来了，当Job2被处理结束后，日志显示，Job2又立马被设置成了INPROGRESS状态，重跑一遍INPROGRESS->COMPLETED。

###### 问题跟踪

跟踪后发现，当Job1在处理的同时，主线程仍然在listStartOrNewEntities，这个时候listStartOrNewEntities的结果只有Job2，所以Job2被不停的挂在线程池等待队列上等待被处理。

这也是为什么Job2在被处理结束状态进入COMPLETED的同时，立即被设置为INPROGRESS的原因。


###### 问题解决

这个问题的解决方案有几种：

方案1： 为JobState增加一个QUEUED的状态。创建CompletableFuture之前，如果状态为QUEUED，就不创建也就避免了排队；

方案2：将INPROGRESS状态设置这个动作，单拎出来，作为一个任务放在两个foreach之间。

方案3：在CompletableFuture的处理函数里面做判断，如果JobState已经为COMPLETED，不予处理直接返回。

其中，方案3，仅仅是跳过处理，仍然让Job进行了等待队列的排队，逻辑不正确，不可取。

只能从方案1和方案2中选择一种解决方案。
