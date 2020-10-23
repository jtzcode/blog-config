---
title: Linux命名空间与容器技术浅谈
date: 2020-10-23 10:35:44
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

`cgroups` (Linux Control Groups) 也是Linux内核的功能，用来限制一个进程的资源使用量，这使得一个进程不能过分使用资源导致其他进程无资源可用。
<!--more-->
## Linux命名空间
Linux系统在默认情况下只有一个命名空间，开发者可以创建额外的命名空间来组织不同的资源，形成不同的**视图**。这里所说的资源指的是操作系统相关的信息（文件、进程、主机等），具体类别如下：
- UTS
- User
- Process ID
- Inter-Process Communication（IPC)
- Network
- Mount

每个进程可以属于不同类型的多个命名空间，表示该进程可以看到所在命名空间定义的系统资源。比如，进程A处于Network命名空间N中，那么A只能看到N内部的网络接口。进程A还可以同时属于UTS命名空间U，这意味着A只能看见U定义的主机名和域名信息。处于同一PID命名空间的进程共享相同的进程树，而只有处于同一IPC命名空间的进程之间才可以进行通信。

每个容器都有一组命名空间，定义各类系统资源的访问范围，身处其中的进程就只能访问容器内部的资源，这就是容器实现的资源隔离。

下图展示了Linux命名空间的父子关系：

![ns](ns-fork.jpg)

可以看到，每个命名空间都来自一个父命名空间，而进程在父子命名空间中可以有不同的进程ID，形成映射关系。

在Linux中，新的命名空间可以通过以下方法创建：

1. 在`fork`或者`clone`系统调用的选项中，指定与父进程共享命名空间，或者建立新的命名空间。
2. 使用`unshare(2)`系统调用将进程的某些部分（如命名空间）从父进程分离。

还可以使用`setns(2)`系统调用让当前进程加入已有命名空间。

下图展示了进程与命名空间在实现上的关系：

![ns](ns-struct.jpg)

每个进程结构都有一个指向`nsproxy`结构对象的指针，而每个`nsproxy`对象定义了一些进程所属的各类命名空间。其中每一类命名空间的定义也是保留一个指针，指向真正的命名空间结构。可以看到，一个`nsproxy`定义可以被多个进程共享，这也意味着修改指定的命名空间，与其关联的所有进程都会受到影响。

内核在fork或者clone进程时会检查如下标志位：`CLONE_NEWNS`, `CLONE_NEWUTS`, `CLONE_NEWIPC`, `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWUSER` 以及 `CLONE_NEWCGROUP`。据此判断是否需要创建命名空间及其类型。

Linux命名空间成为各类容器技术的基础，而后者又在基于容器的各种云平台上广泛运用，比如Kubernetes。掌握Linux命名空间的一些原理和细节，对熟练使用K8S Pods等上层功能十分有益。后面的文章打算从实践的角度继续学习Linux命名空间和容器技术。

## 参考资料
- Kubernetes In Action
- 深入Linux内核架构
- https://man7.org/linux/man-pages/man2/unshare.2.html
- https://man7.org/linux/man-pages/man7/namespaces.7.html
- https://www.linuxjournal.com/content/everything-you-need-know-about-linux-containers-part-i-linux-control-groups-and-process
- https://medium.com/@teddyking/linux-namespaces-850489d3ccf
- https://medium.com/@teddyking/namespaces-in-go-basics-e3f0fc1ff69a