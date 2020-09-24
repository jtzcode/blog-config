---
title: 负载均衡技术概览
date: 2020-09-24 17:38:43
categories:
    - 技术
tags:
    - Tech
    - 后端
    - 服务器
---

## 引言
本文简要介绍负载均衡的一些基本概念、原理和工具，供大家参考。若有实际需求，可进一步查找资料详细了解。本文参考资料：

 1. 书籍：《阿里云运维架构实践秘籍》乔锐杰著
 2. 博文：https://www.cnblogs.com/Javi/p/6483214.html
 3. 官网：LVS: http://www.linuxvirtualserver.org/Documents.html
 4. 官网：Nginx: https://www.nginx.com/
 5. 论文：http://www.linuxvirtualserver.org/ols/lvs.pdf
 6. 博文：https://www.zhihu.com/topic/19607051/hot

 
## 分类
负载均衡根据网络分层可以分为几个类型：
1. 硬件负载均衡：主要在交换机等**网络设备**上实施，通过硬件对网络数据包进行分流和转发，产品有**Citrix Netscaler**和F5。
2. 二层负载均衡：在数据链路层修改数据包，只有 **LVS（Linux Virtual Server）** 支持二层负载均衡。
3. 三层负载均衡：在IP数据包外层添加新的IP数据头，只有**LVS**支持三层负载均衡。
4. 四层负载均衡：根据IP和端口对TCP数据包进行转发，**Nginx**和HAProxy为主流应用。
5. 七层负载均衡：主要通过HTTP协议的数据包进行转发，依据有域名、目录结构，甚至是浏览器类型，Nginx也支持七层负载均衡。
6. DNS负载均衡：将域名匹配多个IP地址，不同客户端返回不同地址，从而实现均衡。该策略使用不多，一个原因是客户端基本都有缓存，使得策略不能生效。

下面就LVS和Nginx的负载均衡实践作简要介绍。
<!--more-->

## LVS 负载均衡
LVS（Linux Virtual Server）是虚拟化的服务器集群，现在已经成为Linux内核的一部分。LVS提供了高效的负载均衡解决方案，并且是唯一支持二层网络负载均衡的技术。

相对于其他负载均衡技术（Nginx、HAProxy等），LVS的优势在于：

 - 对于CPU和内存的消耗极低
 - 集成在内核中，无需复杂的配置
 - 在内核态工作，性能可媲美硬件服务器

### LVS 二层负载均衡
在二层（数据链路）上，LVS的负载均衡需要开启**DR模式**。其工作原理如下（图片来源：参考资料1）：
![LVS DR模式](https://img-blog.csdnimg.cn/20200803151809124.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
客户端请求LVS生成的虚拟IP（VIP），LVS的负载均衡服务器将数据包中的**源MAC地址和目标MAC地址分别修改为自己的MAC地址（DIP）和目标Server的MAC地址（RIP）**。目标服务器接收到发给自己的数据包后，直接将响应发送给**客户端**（而不是LVS）。

这种模式要求目标Server与LVS在同一个局域网（VLAN）中，并且由于目标服务器直接与客户端通信，因此它需要绑定公网的IP。
### LVS 三层负载均衡
在三层（IP）网络上，LVS的负载均衡工作在**TUNNEL模式**上。其工作原理如下（图片来源：参考资料1）：
![LVS TUN模式](https://img-blog.csdnimg.cn/20200803152913737.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
在这种模式下，客户端依然请求LVS的虚拟IP（VIP），这次负载均衡服务器没有修改MAC地址，而是**在IP报文的头部再封装一层IP报文**，把源地址和目标地址分别写为自己的地址和目标服务器的地址。这样就相当于形成一个“隧道”，原来的报文可以在公网上传送，不需要在同一个局域网。

当然，服务器端也需要有**解析两层IP报文**的逻辑，将真正的报文取出，并直接与客户端通信，发送响应。

### LVS 四层负载均衡
在四层（TCP/IP）网络上，LVS负载均衡需要开启**NAT模式**。其工作原理如下（图片来源：参考资料1）：
![LVS NAT模式](https://img-blog.csdnimg.cn/20200803153849899.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
在该模式下，客户端依然请求LVS虚拟IP，LVS负载均衡服务器会将四层报文的目标地址修改为目标Server地址并转发，因此要求LVS与目标服务器处于同一个局域网。

目标服务器**不会直接响应客户端，而是将响应发送给负载均衡服务器**，再由后者与客户端通信。由于请求来回都要通过负载均衡服务器，该模式可能会有性能瓶颈，要慎重使用。

LVS的三种模式的一个特点是，**对于客户端发起的TCP连接，只进行转发，并没有产生新的连接，所有的工作都在内核中完成**，效率很高。而其他的四层负载均衡方案则需要负载均衡服务器建立新的TCP连接到目标服务器，且需要内核态到用户态的切换。

## Nginx 负载均衡
[Nginx](https://www.nginx.com/)是一款优秀的轻量级Web和反向代理服务器，擅长处理高并发的网络请求，可以做四层和七层网络的负载均衡。

使用Nginx进行四层负载均衡的原理如下（图片来源：参考资料1）：
![Nginx 四层负载均衡](https://img-blog.csdnimg.cn/2020080316035462.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
客户端发起的请求中包含了**服务器的IP（DIP）和端口**，Nginx接收到请求会根据转发策略，向目标服务器发起新的TCP连接，指向目标服务器（RIP）的IP和端口。注意，这里**建立了两次TCP连接**，且负载均衡是在**用户空间**接收数据（而非LVS的内核空间）。随后目标服务器将响应返回给Nginx负载均衡，目标服务器与负载均衡服务器也**不需要**在同一个局域网。

对于七层（HTTP）的负载均衡，Nginx的工作原理如下：
![Nginx 七层负载均衡](https://img-blog.csdnimg.cn/20200803161508995.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
这个请求过程与四层负载均衡类似，只不过包含了HTTP的数据包内容，如URL信息。负载均衡服务器依然在用户空间处理请求，并建立新的TCP连接。

七层的负载均衡可以根据域名进行转发，还可以通过URL中路径或者浏览器的类型进行转发，这些信息都可以通过HTTP报文获得。七层负载均衡通常可以用于Tomcat、PHP等Web服务器，四层负载均衡则可以用于MySQL、Redis等数据库服务器。

最后我们来看一个不同层次负载均衡的性能对比，如下图（图片来源：参考资料1）：
![性能对比](https://img-blog.csdnimg.cn/20200803162235478.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)
我们可以看到LVS负载均衡的性能已经可以和硬件设备相媲美，而Nginx等四层、七层的负载均衡方案就要差很多。Web服务器（如Apache）本身提供的负载均衡方案支持并发数就更少了。

## 总结
以上是对一些常见的负载均衡方案的概念和原理进行的简单介绍，具体使用哪种方案还需要根据实际需求，并辅助以正确的配置。想要了解更多细节，可查阅引言中的参考资料。