---
title: Jenkins Pipeline 手记（4）—— 远程Powershell权限困局
date: 2020-09-26 16:49:44
categories:
    - 技术
tags:
    - Tech
    - Jenkins
    - DevOps
    - 日拱一卒
---
## 背景
在Jenkins Pipeline里面，我们经常会使用powershell脚本执行一些必要操作，比如Build项目、部署Service、配置网络等等。有时候我们还需要进行远程的操作，比如当目标机器是另一台虚拟机，或者是一个K8S的节点。本文探讨一个在远程执行powershell命令时遇到的小问题。

## 问题重现
我的项目里有一个需求是，将build出来的service部署到远端的server上。部署的过程其实就是文件拷贝的过程，在部署前需要调用`Get-Service` 判断service是否存在，以及用`Stop-Service`停止正在执行的service。这两个powershell脚本的调用出了问题。

```powershell
Get-Service $service -ComputerName $server -ErrorAction SilentlyContinue | Stop-Service -ErrorAction SilentlyContinue
```
复杂性在于，**这条命令在一个Jenkins的静态slave结点上执行没有问题，而在一个动态分配的slave结点（如K8S Cloud）上执行就会报如下错误**：

```powershell
Get-Service : Cannot open Service Control Manager on computer '10.1.x.x'. This operation might require other privileges.
```
<!--more-->
## 分析与尝试
这看起来是执行这条命令的权限不够，这可以理解，毕竟是在远端机器执行。因此可以得出结论，在静态的slave上应该是配置了针对远端机器的credential信息，而动态slave上显然没有。添加credential的方法如下：

```powershell
cmdkey /add:10.1.x.x /user:admin /pass:password
```
我在部署脚本前执行上面的命令添加credential，并没有效果，问题依旧。由此，推测credential并没有添加成功，或者添加方式有问题。比如，我试了在user前面加上`.\`的域符号，没有效果。

会不会是使用cmdkey的方式在docker container里不生效呢？毕竟K8S的动态节点环境本质还是在Docker container里面。我想试试直接将credential对象传递给脚本方法，而`Get-Service` 方法本身并不支持传递credential，因此我找到了`Invoke-Comand`方法。

```powershell
Invoke-Command -ComputerName 10.1.x.x -Credential (Get-StoredCredential -Target $cred) -ScriptBlock {Get-Service}
```
其中的`$cred`可以通过如下方式生成：

```powershell
Import-Module CredentialManager
$cred = New-StoredCredential -Persist LOCAL_MACHINE -Target cred -UserName admin -Password pass
```
然而，这样调用`Get-Service`会报另一个错：

```powershell
ERROR:  ACCESS IS DENIED
ERROR: The connection to the remote host was refused. Verify that the
WS-Management service is running on the remote host and configured to
listen for requests on the correct port and HTTP URL.
```

通过查看微软的[帮助文档](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting?view=powershell-7)，可以获得这个错误的可能原因。据此，我又尝试了几种方法：

 - 可能是Server端没有开启remote powershell的功能，需要运行：`Enable-PSRemoting`
 - 可能是Server不在Client的信任列表里，尝试执行：`Set-Item WSMan:\\localhost\\Client\\TrustedHosts -Value 10.1.x.x -Force`
 - 可能需要允许使用client credential访问远程机器，尝试运行：`Enable-WSManCredSSP -Role Client -DelegateComputer 10.1.x.x -Force`

这些方法都不能解决拒绝访问，或者开始提到的权限问题。看来另有原因。

## 解决方案
最后我又回归到原点思考，根本原因可能还是credential本身的添加方式有问题，而不是网络通信的问题。

既然password不太可能出问题，那么user的写法呢？我尝试了**使用具体的ip地址作为域名**，取代`.\`，终于解决了问题！

```powershell
cmdkey /add:10.1.x.x /user:10.1.x.x\admin /pass:password
```
注意，如果是Jenkins的powershell插件，命令出现的`\`和`$`都需要转义。

## 结论
一些技术问题，会随着研究的深入而陷入死胡同，或者偏离原来的方向，利正真的解决方案越来越远。当然，深入研究是可取的，不过解决问题有时候才是第一要务。在解决问题过程中，适当地停下来问问自己，我最开始要解决的问题到底是什么？

扩展资料：

 - https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting?view=powershell-7
 - https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/cmdkey