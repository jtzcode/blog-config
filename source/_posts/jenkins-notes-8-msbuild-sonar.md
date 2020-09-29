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
最近工作中有这样的需求，需要针对MSBuild的项目，在pipeline中使用SonarQube进行静态代码分析。这就需要用到 [SonarScanner For MSBuild](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/) 这个版本的扫描工具。如果直接在