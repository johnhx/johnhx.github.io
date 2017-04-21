---
title: Android Handler Leak原理和解决方案
layout: post
categories: Android
tags: Android
---

Android的应用程序开发有一个著名的Handler Leak问题。

在描述这个问题的原理和解决方案前，先摆一个java的基础知识：
>在java中，非静态的内部类和匿名内部类会隐式地持有其外部类的引用。
静态的内部类不会持有其外部类的引用。

基础知识摆了后，下面是正式主题。

关于Android的Hanlder，官方文档有这样一段说明：  
>In Android, Handler classes should be static or leaks might occur, Messages enqueued on the application thread's MessageQueue also retain their target Handler. If the Handler is an inner class, its outer class will be retained as well. To avoid leaking the outer class, declare the Handler as a static nested class with a WeakReference to its outer class.

>翻译：在Android中，Handler类应该是静态的，否则就可能会造成泄漏。应用程序的MessageQueue中排队的Message对象还保留着他们的目标Handler。如果这个Handler是一个内部类，那他的外部类也会被保留（原因见文头）。所以为了避免泄漏外部类，需要将Handler生命为静态内部类，并且让其持有其外部类的WeakReference。

导致泄漏的原因有两个:  
- 原因1: MessageQueue中排队的Message对象持有对Handler的引用导致Handler泄漏;
- 原因2: Handler对象对外部类（如所在的Activity或者Service）实例持有强引用。

这两个原因的共同作用造成了Android的Handler Leak.  

要解决这个问题，就应该从这两个原因入手。  

对于原因1，可以在Handler所在的组件生命周期结束前清除掉MessageQueue中发送给Handler的Message对象.  

对于原因2，可以将Handler实现为静态内部类的方式，从而避免了对外部类的强引用。如果在Handler中需要使用外部类，则可以在Handler内部声明一个WeakReference引用到外部类的实例。