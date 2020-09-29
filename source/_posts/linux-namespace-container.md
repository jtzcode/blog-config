---
title: Linux命名空间与容器技术浅谈
date: 2020-09-26 22:35:44
categories:
    - 技术
tags:
    - Tech
    - 后端
    - 服务器
    - Linux
    - 容器
---
## 前言
本文来谈谈Linux命名空间的机制，以及基于该机制的容器技术是如何实现的。

## 容器技术
我们如果想共享硬件资源，直接的方法是采用虚拟机技术。每台虚拟机可以分配到一定的硬件资源，有独立的操作系统。有时候，我们还想共享操作系统的内核，在更高的层次实现资源的共享和隔离，这就需要容器技术。

以Linux操作系统为例，如果我们想让不同的进程运行在同一个操作系统上，却能看到和使用不同的**系统视图和系统资源**，Linux分别提供了两种实现方法：**Linux命名空间和Linux控制组（cgroups）**。系统视图是指一个进程可以看到的系统相关的信息，如文件、进程、网络接口、主机名等。系统资源则是只具体的硬件资源，如CPU、内存、网络带宽等。

一个容器其实就是**对应不同类型资源的命名空间的集合**，这个集合构成一个系统视图，也可以将容器视为一种轻量级的虚拟机。想要理解容器，首先要理解Linux命名空间。主流的容器技术如Docker，就是基于Linux命名空间实现的。

## Linux命名空间
Linux系统在默认情况下只有一个命名空间，开发者可以创建额外的命名空间来组织不同的资源，形成不同的视图。这里所说的资源指的是操作系统相关的信息（文件、进程、主机等），具体类别如下：
- UTS
- User
- Process ID
- Inter-Process Communication（IPC)
- Network
- Mount

每个进程可以属于不同类型的多个命名空间，表示该进程可以看到所在命名空间定义的系统资源。比如，进程A处于Network命名空间N中，那么A只能看到N内部的网络接口。进程A还可以同时属于UTS命名空间U，这意味着A只能看见U定义的主机名和域名信息。

每个容器都有一组命名空间，定义各类系统资源的访问范围，身处其中的进程就只能访问容器内部的资源，这就是容器实现的资源隔离。
## 参考资料
- Kubernetes In Action
- 深入Linux内核
- https://medium.com/@teddyking/linux-namespaces-850489d3ccf
- https://medium.com/@teddyking/namespaces-in-go-basics-e3f0fc1ff69a
- https://medium.com/@teddyking/namespaces-in-go-user-a54ef9476f2a