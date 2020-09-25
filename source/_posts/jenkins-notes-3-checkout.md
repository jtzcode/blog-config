---
title: Jenkins Pipeline 手记（3）—— 自定义Checkout的陷阱
date: 2020-09-25 15:52:05
categories:
    - 技术
tags:
    - Tech
    - Jenkins
    - DevOps
    - 日拱一卒
---
## 引言
最近在pipeline中checkout代码时遇到了“不可序列化”（**java.io.NotSerializableException**）的问题。这个问题只在特定的场景下能重现，虽然影响不大，但如果深入研究一下，可以加深对Jenkins Pipeline的理解。
## 问题重现
我们知道，在使用Multi-branch pipeline时，可以在job的配置中指定源代码的来源，如git url，credentials，clone options，submodule options等等。然后在pipeline中，可以直接调用：

```bash
checkout scm
```
下载代码。这个scm对象对象由Jenkins的Git插件提供，里面包含了之前配置的源代码的所有信息。然而有的时候，我们也需要对配置好的scm进行订制：
<!--more-->
```python
checkout(
	[
		$class: 'GitSCM', 
	    branches: scm.branches,
	    extensions: scm.extensions + [
	        [$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: true, recursiveSubmodules: true, reference: '', trackingSubmodules: false]
	    ],
	    userRemoteConfigs:scm.userRemoteConfigs
	]
)
```
注意，scm.extensions这个字段后面又连接了一个SubmoduleOption的配置，作为对默认job配置的覆盖。像这样，我们借助默认的scm对象，**重新定义一个scm对象**，传递给checkout方法，就**可能**报序列化的问题:

```bash
java.io.NotSerializableException: hudson.plugins.git.extensions.impl.CloneOption
```
## 原因分析
我通过尝试发现两个问题：

 - 直接checkout scm不会有问题，一旦读取scm，并重新传入，就出问题（哪怕没有改动读取的scm）
 - 我的问题出现在Shared Library中，即在我的library内部调用了checkout方法。如果直接在Jenkinsfile里调用，不会有问题
 
 第一个问题还有个前提，就是我的job配置里指定了Advanced Behaviors，并且设置其中的了**CloneOptions**。这样自动生成的scm对象的extensions字段，才会包含CloneOptions的内容。
 ![CloneOptions](https://img-blog.csdnimg.cn/20200727173431365.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
我查了一下GitClient Plugin的API文档，CloneOptions相关的类确实是不能序列化的，没有实现Serializable接口，而且scm.extensions这个列表本身也是不能序列化的。我们知道，Jenkins Pipeline要求所执行的代码是可序列化的，方便进行CSP风格的转换（详细见：[另一篇文章](https://blog.csdn.net/jtz_MPP/article/details/105425616)），因此就会报错。

如果没有配置CloneOptions，那么序列化问题变为：

```bash
ERROR: java.io.NotSerializableException: hudson.util.DescribableList
```
这就是extensions列表本身的问题。再次强调，这两个序列化错误**只有在读取默认的scm对象再传入checkout方法**时才会出现。

根据这些现象，我认为Jenkins在处理scm对象时，如果是通过plugin默认生成的，它知道如何序列化，或者绕开序列化检查机制（类似@NonCPS的作用）；如果scm被读取，则采用真实的extensions类型，并进行序列化检查，导致异常。事实上，在scm的各个字段中，只有extensions是无法序列化的。

对于上面说的第二个问题，使用场景如下：

```python
library "shared-lib"
def scmSpec = [
  branches: scm.branches,
  ...
]
myFunc(scmSpec)
# myFunc defined in shared-lib
```

至于为什么直接调用checkout scm就不会出错，而在shared library中使用就会出错，我还没有答案。一个猜想是，Jenkins在处理Jenkinsfile和处理嵌套调用的方法时，处理checkout的方式不一样，前者不会进行序列化检查。

关于以上猜想，需要仔细研究Jenkins的CPS Plugin以及GitClient Plugin才能得出结论，目前我还没有着手做。
## 解决方案
值得注意的是，使用 **@NonCPS** 标记并不能解决本文遇到的两个序列化问题。

对于CloneOptions的问题，解决方法是**显式地声明所有extensions内部的选项**例如：

```python 
def scmSpec = [
    $class: 'GitSCM', 
    branches: scm.branches,
    extensions: [
        [$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: true, recursiveSubmodules: true, reference: '', trackingSubmodules: false],
        [$class: 'CloneOption', shallow: false, noTags: false, reference: '', timeout: null, depth: 0, honorRefspec: false]
    ],
    userRemoteConfigs: scm.userRemoteConfigs
]
checkout scmSpec
```
注意，其他scm的字段可以直接读取（例如branches）。这样声明也许会让Jenkins将scm的字段解析成可序列化的方式。

对于DescribableList的问题，可以直接在extensions后面附加一个空的列表，相当于重新显式声明了extensions列表，问题消失。

```python
extensions: scm.extensions + []
```
## 结论
本文讨论了在pipeline中使用checkout时可能遇到的序列化问题陷阱，并给出了workaround的解决方案。如果想彻底解决这类问题，还要研究插件的源代码，理解Jenkins pipeline的运行过程。
有关插件的资料：

 - https://github.com/jenkinsci/workflow-cps-plugin/
 - https://plugins.jenkins.io/git-client/
