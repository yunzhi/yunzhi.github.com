---
layout: post
title: (转载)Android进程间通信（IPC）机制Binder
description: 本篇文章转自csdn博主罗升阳的博客，主要介绍Android进程间通信的主件binder
category:
---

本篇文章转自csdn博主[罗升阳](http://blog.csdn.net/luoshengyang/article/details/6618363)的博客中的关于该主题的一系列文章，主要介绍Android进程间通信的主件binder的实现。

## [Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)

在Android系统中，每一个应用程序都是由一些Activity和Service组成的，这些Activity和Service有可能运行在同一个进程中，也有可能运行在不同的进程中。那么，不在同一个进程的Activity或者Service是如何通信的呢？这就是本文中要介绍的Binder进程间通信机制了。

我们知道，Android系统是基于Linux内核的，而Linux内核继承和兼容了丰富的Unix系统进程间通信（IPC）机制。有传统的管道（Pipe）、信号（Signal）和跟踪（Trace），这三项通信手段只能用于父进程与子进程之间，或者兄弟进程之间；后来又增加了命令管道（Named Pipe），使得进程间通信不再局限于父子进程或者兄弟进程之间；为了更好地支持商业应用中的事务处理，在AT&T的Unix系统V中，又增加了三种称为“System V IPC”的进程间通信机制，分别是报文队列（Message）、共享内存（Share Memory）和信号量（Semaphore）；后来BSD Unix对“System V IPC”机制进行了重要的扩充，提供了一种称为插口（Socket）的进程间通信机制。若想进一步详细了解这些进程间通信机制，建议参考Android学习启动篇一文中提到《Linux内核源代码情景分析》一书。

但是，Android系统没有采用上述提到的各种进程间通信机制，而是采用Binder机制，难道是因为考虑到了移动设备硬件性能较差、内存较低的特点？不得而知。Binder其实也不是Android提出来的一套新的进程间通信机制，它是基于OpenBinder来实现的。OpenBinder最先是由Be Inc.开发的，接着Palm Inc.也跟着使用。现在OpenBinder的作者Dianne Hackborn就是在Google工作，负责Android平台的开发工作。

前面一再提到，Binder是一种进程间通信机制，它是一种类似于COM和CORBA分布式组件架构，通俗一点，其实是提供远程过程调用（RPC）功能。从英文字面上意思看，Binder具有粘结剂的意思，那么它把什么东西粘结在一起呢？在Android系统的Binder机制中，由一系统组件组成，分别是Client、Server、Service Manager和Binder驱动程序，其中Client、Server和Service Manager运行在用户空间，Binder驱动程序运行内核空间。Binder就是一种把这四个组件粘合在一起的粘结剂了，其中，核心组件便是Binder驱动程序了，Service Manager提供了辅助管理的功能，Client和Server正是在Binder驱动和Service Manager提供的基础设施上，进行Client-Server之间的通信。Service Manager和Binder驱动已经在Android平台中实现好，开发者只要按照规范实现自己的Client和Server组件就可以了。说起来简单，做起难，对初学者来说，Android系统的Binder机制是最难理解的了，而Binder机制无论从系统开发还是应用开发的角度来看，都是Android系统中最重要的组成，因此，很有必要深入了解Binder的工作方式。要深入了解Binder的工作方式，最好的方式莫过于是阅读Binder相关的源代码了，Linux的鼻祖Linus Torvalds曾经曰过一句名言RTFSC：Read The Fucking Source Code。

虽说阅读Binder的源代码是学习Binder机制的最好的方式，但是也绝不能打无准备之仗，因为Binder的相关源代码是比较枯燥无味而且比较难以理解的，如果能够辅予一些理论知识，那就更好了。闲话少说，网上关于Binder机制的资料还是不少的，这里就不想再详细写一遍了，强烈推荐下面两篇文章：

[Android深入浅出之Binder机制](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)

[Android Binder设计与实现 – 设计篇](http://disanji.net/2011/02/28/android-bnder-design/)

Android深入浅出之Binder机制一文从情景出发，深入地介绍了Binder在用户空间的三个组件Client、Server和Service Manager的相互关系，Android Binder设计与实现一文则是详细地介绍了内核空间的Binder驱动程序的数据结构和设计原理。非常感谢这两位作者给我们带来这么好的Binder学习资料。总结一下，Android系统Binder机制中的四个组件Client、Server、Service Manager和Binder驱动程序的关系如下图所示：

![android binder system](/images/binder/android_binder_system.png)

1. Client、Server和Service Manager实现在用户空间中，Binder驱动程序实现在内核空间中

2. Binder驱动程序和Service Manager在Android平台中已经实现，开发者只需要在用户空间实现自己的Client和Server

3. Binder驱动程序提供设备文件/dev/binder与用户空间交互，Client、Server和Service Manager通过open和ioctl文件操作函数与Binder驱动程序进行通信

4. Client和Server之间的进程间通信通过Binder驱动程序间接实现

5. Service Manager是一个守护进程，用来管理Server，并向Client提供查询Server接口的能力


## [浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)


["Yunzhi made"](http://yunzhi.github.io) &copy;
