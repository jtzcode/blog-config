---
title: Jenkins Pipeline 手记（6）—— 使用 Azure VM 作为动态节点
date: 2020-09-26 17:05:06
categories:
    - 技术
tags:
    - Tech
    - Jenkins
    - DevOps
    - 日拱一卒
---
在Jenkins Pipeline实践中，有时需要分配并连接动态的build agent完成CI任务。相比于静态的VM节点，这样做的好处是可以优化资源的使用，动态地分配和释放所需要的物理资源，既不让资源闲置，也不会导致某些节点负载过重。

Jenkins支持通过Cloud的方式配置动态节点，比如Kubernetes和Azure VM。本文就来分享一下如何配置并使用Azure VM来构建CI任务。

## 准备工作
首先，你要有一个Azure Subscription和Service Principal。使用Azure Portal 的创建方法可以参考[这里](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal)，使用Powershell的创建方法可以参考[这里](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-authenticate-service-principal-powershell)。

简单来说，如果使用Azure Portal方式，需要先在一个Subscription下面注册一个Web Application，随后一个service principal会自动创建出来。一个service principal定义了**一个App在特定的tenant里面能做什么，能够访问哪些资源，以及能够被谁访问**。
<!--more-->
在Jenkins中添加一个`Microsoft Azure Principal`类型的Credential，添加刚刚新建的Principal信息。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907110650988.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)

## 基本原理
在Jenkins中使用Azure VM需要安装插件：[Azure VM Agents Plugin](https://plugins.jenkins.io/azure-vm-agents/)。 当一个Build开始时，上述插件会在Azure上创建一台虚拟机，然后连接到Jenkins Master实例，Build任务就会在这个虚机上运行。虚拟机的硬件配置以及预装的软件和工具可以通过模板和脚本来配置，后面会谈到。

这台虚机会保留几个小时（可配置），后续的一些Build任务可以继续重用这台虚机，这样就会省去创建新虚拟机的时间。一旦一台虚拟机空闲一段时间（可配置），这个Build 节点就会被删除，以释放计算资源。

更多的原理可以参考配置部分。
## 配置
Jenkins的管理员可以配置插件。具体入口是：**Manage Jenkins -> Configure System ->Add a new cloud -> Microsoft Azure VM Agents**。基本配置界面如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020090713412751.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
其中`Azure Credentials`一栏就填写刚才新建的Service Principal的信息。Resource Group就填Azure上面所在Subscription的Resource Group，推荐填已有的Group，防止输入出错。此外，还可以配置最大agent的数量和部署超时时间等信息，这可以根据自己的业务逻辑和资源需求来决定。

下面就是模板（template）的配置。模板定义了Azure虚拟机的默认的配置，比如大小、存储的类型和位置等等。模板的定义如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907140107918.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
`Agent Workspace`就对应Jenkins agent的Workspace的概念，这个路径可以是个临时的，因为我们假设Workspace不会存储重要数据。`Labels`就是节点的label，在pipeline里面node函数中使用的就是这个Label，推荐填写有意义的标签名。

Storage Account需要在Azure Subscription中创建，用于存储VM的OS磁盘。这里指定Storage Account的类型一般选择`Standard_LRS`，尤其是在下面的Disk Type配置选择`Managed Disk`时，这可以避免过大的磁盘开销。

Rentention Strategy定义了VM过期的策略，默认是在一定时间后，关掉并删除**空闲的**虚拟机，还支持其他两种策略，可以参考插件的文档。超期时间默认是60分钟，可以根据需要改动。 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907141712630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
接下来是选择虚拟机OS的镜像，有默认的系统镜像可选，比如Windows Server 2016、Ubuntu 16.04等。可以自定义预装的常用软件如Git、Maven和Docker。最后，添加虚拟机的管理员账户的Credential，可以事先在Jenkins master上定义好。

注意，OS镜像时也支持自定义的模板，这是比较常见的需求。因为我们的**Build任务很可能需要特定的系统和软件工具**。选择`Use Advanced Image Configurations`：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907142333610.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
对于Image的类型，可以选择`Custom User Image`，这里的Image要能被之前配置的Storage Account发现。还可以选择`Custom Managed Image`，这里的Image ID就是Azure 托管的image，类似`/subscriptions/{GUID}/resourceGroups/baseimages-master/providers/Microsoft.Compute/images/test`。当然可以指定其他来源的Image，具体配置方法参考资料1。

Launch Method定义了与agent的连接方式，支持**SSH**或者**JNLP**。对于Linux节点只支持SSH的方式，对于Windows节点配置为SSH时，需要勾选预装SSH Server的选择。官方**推荐使用SSH**的连接方式，因为JNLP的方式需要更多的初始化工作：

```bash
When using the JNLP launch option, ensure the following:

- Jenkins URL (Manage Jenkins -> Configure System -> Jenkins Location)
- The URL needs to be reachable by the Azure agent, so make sure to configure any relevant firewall rules accordingly.
- TCP port for JNLP agent agents (Manage Jenkins -> Configure Global Security -> Enable security -> TCP port for JNLP agents).
- The TCP port needs to be reachable from the Azure agent launched using JNLP. It is recommended to use a fixed port so that any necessary firewall exceptions can be made.

If the Jenkins master is running on Azure, then open an endpoint for "TCP port for JNLP agent agents" and, in case of Windows, add the necessary firewall rules inside virtual machine (Run -> firewall.cpl).
```
其核心就是要保证Jenkins URL和JNLP的TCP端口对于Azure VM来说时可达的，这可能设计端口和防火墙的配置。

最后来看看初始化脚本的配置。初始化脚本会在虚拟机刚分配出来时执行，用于预先对系统做一些配置，以及预装一些有用的软件。比如，如果你的image里面没有提供Java，则需要在初始化脚本中把Java装好。一个采用JNLP连接方式的Windows脚本示例如下：

```powershell
Set-ExecutionPolicy Unrestricted -Force
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
$jenkinsserverurl = $args[0]
$vmname = $args[1]
$secret = $args[2]
  
# Downloading jenkins slaves jar
mkdir c:\jenkins
$slaveSource = $jenkinsserverurl + "jnlpJars/slave.jar"
$destSource = "c:\jenkins\slave.jar"
$wc = New-Object System.Net.WebClient
$wc.DownloadFile($slaveSource, $destSource)
  
# Download the service wrapper
$wrapperExec = "c:\jenkins\jenkins-slave.exe"
$configFile = "c:\jenkins\jenkins-slave.xml"
$wc.DownloadFile("https://github.com/winsw/winsw/releases/download/v2.9.0/WinSW.NET4.exe", $wrapperExec)
$wc.DownloadFile("https://raw.githubusercontent.com/Azure/jenkins/master/agents_scripts/jenkins-slave.exe.config", "c:\jenkins\jenkins-slave.exe.config")
$wc.DownloadFile("https://raw.githubusercontent.com/Azure/jenkins/master/agents_scripts/jenkins-slave.xml", $configFile)
  
# Prepare config
$configExec = "java"
$configArgs = "-jnlpUrl `"${jenkinsserverurl}computer/${vmname}/slave-agent.jnlp`" -noReconnect"
if ($secret) {
    $configArgs += " -secret `"$secret`""
}
(Get-Content $configFile).replace('@JAVA@', $configExec) | Set-Content $configFile
(Get-Content $configFile).replace('@ARGS@', $configArgs) | Set-Content $configFile
(Get-Content $configFile).replace('@SLAVE_JAR_URL', $slaveSource) | Set-Content $configFile
  
# Install the service
& $wrapperExec install
& $wrapperExec start
```
初始化脚本推荐在`Deployment Timeout`配置指定的时间内结束（默认20分钟），不建议执行过于复杂的逻辑。对于比较重的任务结果可以直接build到image中，然后直接从这个image启动。

在模板的高级选项里也有一些有用的配置，比如虚拟网络。默认情况下，VM agent不属于任何虚拟网络，Jenkins master可以通过插件生成的一个公网IP访问到VM agent。为了安全考虑，也可以不生成这个公网IP，但是要保证Jenkins master和VM agent位于同一个虚拟网络中。

另外还可以通过`Number of Executors`来配置一个VM可以并发运行的build个数，以最大化利用资源。该数值乘以agent的数量就是能同时运行的总的Build数量。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020090715221475.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)


对于其他配置的详情，可以参考资料1以及Jenkins配置页面的提示内容。（**点击配置项后面的问号图标**）
## 运行
在pipeline中运行一个Build比较简单，如：

```groovy
pipeline {
    agent {
        node {
            label 'azure-windows2019'
        }
    }
    stages {
        stage('Build') {
            steps {
                powershell 'Write-Host "Hello Azure VM!"'
            }
        }
    }
}
```
其中`azure-windows2019`就是配置好的Azure VM的label，所有其他的事情就交给插件来完成。

值得一提的是，以上所有的配置工作都可以通过Groovy脚本自动化的完成，即**“Configuration-as-Code**"。详细的代码请参考资料1的“**Configure VM Template using Groovy Script**”一节。
## 参考资料

 - https://plugins.jenkins.io/azure-vm-agents/
 - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal
 - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-authenticate-service-principal-powershell
 - https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals
