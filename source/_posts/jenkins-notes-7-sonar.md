---
title: Jenkins Pipeline 手记（7）—— 发现 SonarQube Plugin 的 Bug
date: 2020-09-26 17:08:03
categories:
    - 技术
tags:
    - Tech
    - SonarQube
    - Jenkins
    - DevOps
    - 日拱一卒
---
今天跟大家分享一个Bug的发现之路，由于这个应用场景比较特殊，其中的分析过程我觉值得记录下来。
## 背景
我们知道，SonarQube 是常用的静态代码扫描工具，可以帮助我们发现各种编程语言编写的项目中的代码问题。针对MSBUILD的项目，SonarQube还提供了专门的扫描工具**sonarscanner-msbuild**。

想要在Jenkins Pipeline中使用这个工具，我们当然可以直接在命令行中调用，并传入需要的参数。但由于参数中有一些常用变量比如server的地址、项目名称等，写死在命令行中不是好办法，可以通过配置的方式提高灵活性。而且，类似sonar.login的参数包含敏感信息，不宜写在命令行中。因此，使用Sonar Scanner的Jenkins 插件是常见的做法（参考资料3）。

很多时候，为了重用Build资源，我们会在Docker container中完成大部分CI任务，Pipeline结束后就可以释放Build的资源留作他用。比如使用Azure VM作为Jenkins的Build Agent时，每个虚拟机的资源在空闲时会释放掉，也无需安装任何Build工具，只需要装好Docker，然后启动container完成Build。（有关Azure VM的用法可以参考[另一篇博文](https://blog.csdn.net/jaytalent/article/details/108441952)。）
<!--more-->
## 重现
我遇到的问题基于这样的场景：需要Build的项目基于.NET Framework 4.8，使用MSBuild进行build。在Pipeline中我会启动并连接一台VM，然后在VM中启动事先指定的Docker container。通过`-v`指令将VM上的代码根目录装载到container中，然后在container内部执行Build、Test、Sonar Scan、Publish等操作。

在执行Sonar Scan时，我使用的`SonarScanner.MSBuild.exe`会报错：

```bash
Failed to parse properties from the environment variable 'SONARQUBE_SCANNER_PARAMS'
```
这是在msbuild sonar scan的`begin`阶段就出错了，因此Build还没开始，Pipeline就因为这个错误停止了。
命令如下：

```bash
SonarScanner.MSBuild.exe begin /k:test /n:test /d:sonar.host.url=https://test-sonarqube.citrite.net /d:sonar.verbose=true /d:sonar.login=****** /v:23 /d:sonar.branch.name=windows-test /d:sonar.branch.target=master
```
我的同事在另一台Jenkins上Build类似的项目，使用同样的场景，却没有遇到这个问题。于是我进行了一番分析。
## 分析
### 1. 定位 `SONARQUBE_SCANNER_PARAMS`
我首先比较了失败和成功Build的Log。发现我的Log里面多了这样一些内容：

```bash
SonarScanner for MSBuild 4.5
15:00:03  Using the .NET Framework version of the Scanner for MSBuild
15:00:03  Default properties file was found at C:\jenkins\tools\hudson.plugins.sonar.MsBuildSQRunnerInstallation\MSBuild-Scanner-BVT-48\SonarQube.Analysis.xml
15:00:03  Loading analysis properties from C:\jenkins\tools\hudson.plugins.sonar.MsBuildSQRunnerInstallation\MSBuild-Scanner-BVT-48\SonarQube.Analysis.xml
15:00:03  sonar.verbose=true was specified - setting the log verbosity to 'Debug'
15:00:03  Pre-processing started.
15:00:03  Preparing working directories...
15:00:03  Using environment variables to determine the download directory...
15:00:03  Loading analysis properties from C:\jenkins\tools\hudson.plugins.sonar.MsBuildSQRunnerInstallation\MSBuild-Scanner-BVT-48\SonarQube.Analysis.xml
```
看起来是sonar的plugin在读取一些配置信息，先检查XML配置文件，再读取环境变量。这段Log之后就出现了上面的错误。由于错误信息告诉我们是在解析环境变量`SONARQUBE_SCANNER_PARAMS`时出了维问题，于是我首先定位一下这个环境变量。

结果是，我自己的Pipeline代码里没有设置过这个变量。而在sonar scanner插件的代码中**设置**了这个变量，在sonar scanner本身的代码中**读取**了这个变量。解析错误很可能是由于，读取的环境变量为JSON字符串，但出于某种原因**传入解析函数的JSON字符串不是合法的**。

先不说解析的问题，为什么我的Build会多出上面那些Log呢？一定有什么机制触发了这段逻辑，只要避免它， 至少能绕开这个问题。

### 2. 检查Sonar的配置
结合Log，我比较了失败和成功Build对Sonar的配置。我发现成功的Build使用是Jenkins自动安装的sonar scanner（可以在Global Tools里面配置），而我的Build直接调用了Container中的sonar scanner。于是我采用了相同的做法，但没有用。**无论是使用Jenkins的还是container的scanner** ，错误依然存在。

我还发现成功项目在SonarQube server上已经存在，因此没有进行全扫描，而我的是个新项目。我担心这也会影响sonar scanner插件的逻辑，于是事先建好了项目，但也没起作用。

### 3. 检查Scanner的版本
下一个猜测是，会不会跟Scanner的版本有关系？成功的Build使用的针对MSBuild 4.8 的scanner，我使用的是针对MSBuild 4.5的，可能旧版本的scanner与插件有兼容问题。于是我更新到了相同的版本，但问题依然复现。

### 4. 其他工具的干扰
我重新分析Log发现，成功的Build在执行sonar前会执行一个[Snyk](https://snyk.io/)工具，该工具会检查代码依赖的第三方库有什么安全隐患，最后一些检查结果也会上传到SonarQube。也许该工具也会影响Sonar的行为？我试着在自己的Build也启动Snyk扫描，但并没有效果。那段Log依然出现。

### 5. Pipeline的代码结构
接着我又审视了自己Pipeline代码的结构。我的这个pipeline有些特殊，首先定义了一个标准的pipeline，然后在其中的某个stage又加载了另一套pipeline的代码，可以理解为pipeline的嵌套。而我的错误就是出现在嵌套的pipeline中。

这里面有一些环境变量的设置，会不会干扰了Sonar的行为？为了排除可能，我简化了pipeline的定义，只在最外层的pipeline调用sonar，问题依旧。
### 6. 不使用Container
我突然想到，如果不用Docker，而只使用静态的节点去Build结果怎么样？于是我分别在Azure VM和一台普通的虚拟机节点上分别跑了一次Build，那个问题不见了，Sonar可以正常执行。这说明那个问题可能**跟Docker container的环境变量传递**有关。
### 7. 使用本地Docker
既然可能与Docker相关，我先在本地试试。我在开发机上起了一个Docker container，然后将项目目录装载到container中，在该目录下执行sonar scanner，结果没有问题。而且，第1节中提到的那些log也出现了，但并没有因为环境变量出错。

结合6和7的结果，我可以判断问题**出现在Jenkins的sonar scanner插件**中，由于Docker的使用导致该插件设置了那个可疑的环境变量`SONARQUBE_SCANNER_PARAM`。
### 8. 更新Sonar 插件
于是我再次回顾插件的代码，发现在[某个时候](https://github.com/jenkinsci/sonarqube-plugin/blob/master/src/main/java/hudson/plugins/sonar/SonarBuildWrapper.java#L162)的确设置了这个环境变量：

```java
map.put("SONARQUBE_SCANNER_PARAMS", sb.toString());
```
再次之前构造这个JSON字符串的确考虑了每个字段名的双引号，但是在后续的使用中，这个双引号被未知原因拿掉了，导致在sonar scanner中解析出错。

具体是哪里导致的问题我还没有找到，目前的workaround是把上面的代码注释掉，因为Sonar并没有使用这个环境变量。然后重新编译插件，更新到Jenkins上，问题暂时解决。
## 后续
作为后续工作，我还要继续考察这个Bug的真正出处。一种可能是sonar scanner本身的代码有问题，是它在读取环境变量时，把双引号丢掉了，不过还需要进一步研究。
## 参考资料

 - https://www.sonarqube.org/
 - https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/
 - https://docs.sonarqube.org/latest/analysis/jenkins/
 - https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-jenkins/
 - https://blog.csdn.net/jaytalent/article/details/108441952
 - https://github.com/jenkinsci/sonarqube-plugin/blob/master/src/main/java/hudson/plugins/sonar/SonarBuildWrapper.java#L162
