---
title: Jenkins Pipeline 手记（5）—— Docker In Docker 那些事儿
date: 2020-09-26 17:00:55
categories:
    - 技术
tags:
    - Tech
    - Jenkins
    - DevOps
    - 日拱一卒
---
## 引言
在定义Jenkins CI Pipeline时，我们难免会用到Docker。将复杂Build/Test/Deploy的过程放到容器中，既可以重用环境设置和工具集，也能高效地使用资源。不过我们可能遇到如下场景：

 - 在build过程中需要使用Docker命令来启动新的容器或者build新的image
 - Jenkins的master或者slave本身就是在容器中运行的

无论哪种情况，本质都是在Docker容器内部运行Docker命令，即所谓的 **Docker-In-Docker - DIND**场景。本文简单介绍DIND的概念和应用。

## 概念和原理
Docker In Docker从原理上说实际有两种方式，第一种可以说是**伪DIND**，也就是Docker的client端在另一个Docker容器中，而server端（docker daemon）还是在宿主机host上。
<!--more-->
比如，你在host上启动了一个container（container-with-docker），这个container里面装有Docker CLI，你可以进行docker ps/push/build等操作：

```bash
docker run -it container-with-docker sh
```
接下来在container的sh程序里，你需要执行`docker images`命令。直接执行是不行的，因为命令的执行需要与Docker的server端daemon进程通信，container-with-docker这个容器内部并没有这个进程。这就引出了DIND的第一种方式：

```bash
docker run -it -v /var/run/docker.sock:/var/run/docker.sock container-a
```
这里使用了`docker run`的`-v`参数，把docker宿主机的docker.sock套接字装载到目标container中，UNIX socket实现为文件，因此可以用来装载。不过这里要求启动的container是一个linux container（比如名为container-a的container）。

这样，container内部就可以执行docker命令了。不过在container内部的**Docker CLI仍然在于Docker宿主机的daemon通信**。并且，如果在内部新启动一个container-b，它与原来的container-a是**兄弟（sibling）关系，而非父子（children）关系**，也就是在宿主机上使用`docker ps`可以看到新的container。如下图（图片来自参考资料1）：
![Docker outside of Docker](https://img-blog.csdnimg.cn/20200814164509658.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
因此，这并不是真正的Docker In Docker，因为内部启动的container并没有在内部运行，也没有与container内部的daemon通信。这种情况常被成为**Docker outside of Docker（DOOD）**

那么如何实现真正的DIND呢？

实际上有优秀的开发者（JÉRÔME PETAZZONI）已经找到解决方案并开发出相应的image。相关代码和文档请参考资料2和3。

总的来说，DIND的思路是在内部container里面也装有daemon process并可以启动。然后，内部的container就是在内部运行，并通过CLI与内部的daemon process通信（而非外部host的daemon）。用法如下：

```bash
docker run --privileged --name my-dind -it -d docker:dind
```
首先启动一个装有完整Docker的container（docker:dind)，注意要使用`--privileged`开关，否则会有权限问题。如果你的Docker版本不支持这个开关，则需要下载最新版本。

`-d`参数让Docker daemon在后台运行，不进入任何程序。使用`--name`选项指定一个名字，稍后使用。

下面再启动一个新的container “docker”，并链接到（link）到刚才container（`my-dind`）的daemon：

```bash
docker run -it --rm --link my-dind docker sh
```
启动成功后进入shell，我们可以在这个container里面运行Docker命令了：

```c
/ # docker info
```
结果如下：

```bash
Client:
 Debug Mode: false

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 19.03.12
 Storage Driver: overlay2
  Backing Filesystem: xfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 7ad184331fa3e55e52b890ea95e65ba581ae3429
 runc version: dc9208a3303feef5b3839f4323d9beb36df0a9dd
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 3.10.0-1062.18.1.el7.x86_64
 Operating System: Alpine Linux v3.12 (containerized)
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 7.629GiB
 Name: 112c6d7e818a
 ID: K5MJ:G3VS:IPWZ:JSW5:YJ3E:KQ5H:L7GA:TCLZ:5XXI:WV7E:YL5I:N3IJ
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
 Product License: Community Engine

```
如果运行命令时出现如下错误：

```bash
error during connect: Get http://docker:2375/v1.40/containers/json: dial tcp: lookup docker on 10.108.x.x:53: server misbehaving
```
说明Docker期待与2375端口通信，但是没有找。事实上，DIND最新版里由于支持TLS，也打开了2376端口，并且是默认行为，Docker其实在与2376端口通信。想要关闭这个默认功能，可以如下这样启动后台的daemon：

```bash
docker run --privileged --name mine-dind -e DOCKER_TLS_CERTDIR="" -d docker:dind
```
通过把`DOCKER_TLS_CERTDIR`环境变量设为空，让DinD在非TLS模式下工作，从而找到正确的端口。有关这个开关的具体信息请参考资料3。

最后多说几句，DIND并不是推荐的最佳实践，其作者专门写了一篇文章（资料4）来分析DIND的一些缺陷，在使用前请确认你真的需要，且没有其他替代方案。

另外，DIND并不支持Windows container，即使是使用DOOD方案，也要保证内部的container是linux container（因为基于Unix Socket），尽管host可以是Windows 版本的Docker。

## 参考资料

 1. https://blog.nestybox.com/2019/09/14/dind.html
 2. https://www.docker.com/blog/docker-can-now-run-within-docker/
 3. https://hub.docker.com/_/docker/
 4. https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/
 5. https://itnext.io/docker-in-docker-521958d34efd
 6. https://github.com/jpetazzo/dind
 
