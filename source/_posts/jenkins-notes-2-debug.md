---
title: Jenkins Pipeline 手记（2）—— 调试小技巧
date: 2020-09-25 15:49:33
categories:
    - 技术
tags:
    - Tech
    - Jenkins
    - Debug
    - DevOps
    - 日拱一卒
---
### 引言
最近的工作中使用Jenkins进行CI的开发和维护，经常需要调试写好的**Jenkinsfile**。然而，每次小的改动都需要提交代码，然后push到远端，Jenkins master读取新版本的Jenkinsfile，查看效果。

这样做一来比较麻烦，尤其是频繁改动或者加一些测试代码的时候。另外，有一些feature branch是大家共同开发维护的，经常提交改动会触发不必要的job build，浪费资源，也影响其他人开发。这篇文章以此为契机分享一些跟Jenkins调试相关的小技巧。

### Jenkins Pipeline
首先需要明确一点，我使用的Job类型是**Pipeline**，顾名思义，这类Job用于构建一些CI的流水线，在流水线上可以完成一系列的操作，诸如Build，Unit Test，静态代码扫描，打包，上传到Archive，签名，部署等等。在Pipeline上完成的工作比较多，也比较成体系。
<!--more-->
还有一种Pipeline称为**multi-branch pipeline**，顾名思义，在这个job的配置中，可以指定来自多个repository的branch。通常一个项目的代码的多个branch都需要CI的工作时，可以采用这个类型。这里不多做介绍，大家可以参考Jenkins官方文档。
![Jenkins Pipeline](https://img-blog.csdnimg.cn/20200501143531603.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70)
每个Pipeline job是通过一个名为**Jenkinsfile**的文件驱动的。这个文件通常位于你项目代码的根目录（如果是multi-branch pipeline，则每个branch都需要有这个文件）。不过，也可以在job的配置中指定这个文件的路径。Jenkinsfile可以使用声明式的语法写，也可以用Jenkins支持的脚本语言（如Groovy）去写，我个人比较倾向于后者，功能比较强大。

这就回到开篇的问题了，如果我在开发中不停地更改这个Jenkinsfile，就需要反复提交代码触发build，并查看log。有没有比较简捷的方式呢？

### Build Replay
Jenkins的build页面有个replay的功能，它能够加载你上一次build的Jenkinsfile，然后呈现在页面上供你在线修改，之后可以直接以新的Jenkinsfile重新运行上一次的build。很惭愧我至今才发现这个功能。这样，一些简单的调试功能就可以在线上完成，不需要反复提交代码。当调试结束，所有代码功能正常时，再一次性提交。
![Build Replay](https://img-blog.csdnimg.cn/20200501143113857.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70)
有时候我们会有这样的需求，当脚本执行到某一步时，需要暂停一下，去查看某些系统的状态。比如，我想看看Docker image是否上传成功，或者Sonar Scanner的结果是否生成。在使用Replay功能时，可以通过指定input指令做到这一点。
![使用Input指令](https://img-blog.csdnimg.cn/20200501143240342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70)
### Pipeline Steps
查看log是一种常见的问题排查方式。然而，如果你的Job执行的任务很多，build生产的log量也比较巨大，那查看完整的log页面的相应速度会很慢，体验会很差。

另外，我们在写Jenkinsfile时会不免使用并发操作，比如Parallel语句块。那么这些并发操作产生的log就会互相纠缠在一起，调试和排查时也会很痛苦。解决方法有两种：

一是使用**Blue Ocean**的功能。这个页面会根据Job中不同的阶段来展示相应的log，并发的log也会区分开来。你可以点击相应的Stage图片，查看需要的log。不过当你的log量巨大时，Blue Ocean的打开也会很慢。
![Blue Ocean](https://img-blog.csdnimg.cn/2020050114380789.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70)
二是使用**Pipeline Steps**。这个页面会将你的log组织成**树状结构**。树的层次结构大概与代码中的层次结构对应，并且并发操作的log会组织到各自的调用层次中，不会混乱。通过每一个节点后面的表示状态的图标来判断是否有错误发生。这个页面的响应速度不错，当你的log页面打不开时，可以尝试这个方法。

![Pipeline Steps](https://img-blog.csdnimg.cn/20200501144018397.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70)
### 远程调试
我一直好奇Jenkins有没有远程调试的方案。类似Visual Studio那样，通过一个代理的服务器，远程调试Azure Cluster上的代码。在Google上找了一圈，貌似没有比较好用的方案。在一些IDE的插件里面也找了找，没有什么收获。

但我觉得这应该是个比较普遍的需求: 需要知道Jenkins中运行的代码（包括Script和Shared Library）是否正确执行。Replay虽然可以简单调试Jenkinsfile，但不能加断点，对间接引用的库也无能为力。

参考资料2有一个方案，不过操作起来比较繁琐，需要自己搭建服务器，也只适用于特定场景，大家可以参考。如果有人知道比较好的调试方案，欢迎留言，谢谢。

### 参考资料

 1. https://notes.asaleh.net/posts/debugging-jenkins-pipeline/
 2. https://www.gee-whiz.de/2018/remote-debugging/
 3. https://www.jenkins.io/doc/book/pipeline/development/#replay
