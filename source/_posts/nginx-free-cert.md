---
title: 如何为个人HTTPS站点申请免费证书
date: 2022-03-04 17:01:38
categories:
    - 技术
tags: 
    - 技术
    - nginx
    - 证书
    - HTTPS
---
## 介绍
我们可能都搭建过个人博客站点，在申请了云主机和域名后，可以起一个Web服务器（如nginx）用于host静态的文本和图片，然后就可以通过域名（需备案）访问了。不过默认情况下，站点还是基于HTTP的，没有SSL证书，因此无法通过HTTPS访问。正好最近在极客时间学习陶辉老师的nginx课程，总结了为站点申请免费SSL证书的方法，以及遇到的问题。

### 前提
应用本文的方法有几个前提：
1. 已经为主机申请了域名，IP地址不适合。
2. 采用Nginx作为Web服务器。
3. 可以接受证书临时有效。

### 安装必要的module
首先，你的nginx服务器需要开启http_ssl_module。这需要重新编译nginx，在nginx目录的configure脚本中
### 解决系统依赖
### 安装certbot
### 配置nginx插件
### 修改nginx配置

## 参考
[1] Nginx课程
[2] certbot
[3] 