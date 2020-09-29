---
title: Jenkins Pipeline 手记（8）—— 踩坑 Docker + MSBuild + SonarScanner
date: 2020-09-29 09:45:41
categories:
    - 技术
tags:
    - Tech
    - SonarQube
    - Jenkins
    - DevOps
    - 日拱一卒
---
## 问题
最近工作中有这样的需求，需要针对MSBuild的项目，在pipeline中使用SonarQube进行静态代码分析。这就需要用到 [SonarScanner For MSBuild](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/) 这个版本的扫描工具。如果直接在Jenkins的虚拟机节点上运行Build和Sonar Scan，没有问题。不过我们的Build是在Docker container中运行的，pipeline会启动一个windows container，其中包含了MSBuild和msbuld-sonarscanner等工具。在Sonar阶段就会遇到以下问题：

![error](error.png)
<!--more-->
从Log中的`possible causes`可以看出，原因可能有四种。这里要补充一下MSBuild SonarScanner的一些背景，这个工具分三步执行：`Begin -> Build -> End`。首先运行sonar scanner的`begin`阶段，做一些预处理的工作，生成必要的临时目录，读取配置文件。然后进入项目的Build阶段，此时会调用MSBuild工具去Build某个solution或者project。最后是Sonar Scanner的`End`阶段，这里会真正扫描并分析代码。

## 分析
MSBuild项目特殊的地方就在于扫描分析依赖于Build过程，正如上面第1点建议所说，Build必须在Begin和End步骤之间进行。我接着检查了其他三点，发现只有第三点是可疑的，即**三个步骤需要在同一个目录下进行**。

我在CI中使用了自己定义的一个pipeline模板，该模板在三个不同阶段会分别启动container完成任务，不过我在三个阶段使用了相同的container，且都在container启动时将当前workspace的路径mount到container相应的路径，因此可以保证三个阶段是在同一路径下进行的。那么问题的原因是什么呢？

于是我进一步分析一下每个阶段的Log，比如：
```shell
$ docker run -d -t -v C:\jenkins\:C:\jenkins -w C:/jenkins/workspace/...
```
每次Jenkins的Docker插件都会通过上面的命令启动container，并装载当前的workspace。由于指定了`-d`参数，container是在后台运行的，后续应该还会使用`exec`命令执行需要的操作。在Build或者Sonar执行结束后：
```shell
$ docker stop --time=1 48d4XXX
$ docker rm -f 48d4XXX
```
即每次都会退出并删除container。问题可能就出在这里，每次container退出，那么一些命令执行的上下文就失效了。不过由于我们装载了workspace，凡是在当前workspace改动的目录和文件就得到了保持。那么问题应该出现在workspace以外的路径。

我又尝试着在同一个container里面运行了begin和build，以及build和end阶段，发现**begin和build必须在同一个container，而end可以在新的container中**。这就说明，MSBuild SonarScanner在Begin阶段写了workspace以外的目录，然后在Build阶段会被MSBuild工具读取。

在本地执行sonar scan的Log印证了我的猜想：

![sonar](sonar.png)
Scanner在进行预处理时会**将跟SonarQube相关的targets文件写到当前User的Local目录下**，而这些目录是没有被Docker装载的，且MSBbuild在执行时会读取，并运行相应的task：

```shell
SonarQubeCategoriseProject:
  Sonar: (msBuildTests.csproj) Categorizing project as test or product code...
  Sonar: (msBuildTests.csproj) project has the MSTest project type guid -> test project
  Sonar: (msBuildTests.csproj) Project categorized. SonarQubeTestProject=true
CreateProjectSpecificDirs:
  Creating directory "E:\Projects\test\.sonarqube\conf\1".
SonarQubeImportBeforeInfo:
  Sonar: (msBuildTests) SonarQube.Integration.ImportBefore.targets was loaded
```
而如果在全新的container运行MSBuild，就不会出现上述的Log，紧接着在End阶段就会出错。
## 解决
## 参考资料
- https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/