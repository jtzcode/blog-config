---
title: Web 键盘输入法应用开发指南——输入法事件
date: 2022-01-29 09:59:54
categories:
    - 技术
tags: 
    - 技术
    - 输入法
    - Web
    - JavaScript
---
## 介绍
在这篇文章中，我们开始探讨浏览器对输入法（IME）<sup>[1]</sup>相关事件的支持。我们经常使用中文，因此对输入法并不陌生。事实上，输入法只有在中文、日文和韩文等少数语言中有用，大部分欧美人可能都没有输入法的概念。不过考虑到使用输入法的人群的绝对数量还是很大的，各大浏览器厂商对输入法事件的支持也相对完善。

## 基本概念
以中文拼音输入法为例，输入的过程大致可以分为**组字（composition）**和**提交（commit）**两阶段。比如我们想打“你好”两个字，会在输入框输入“nihao”的拼音，当输入第一个字母“n”时，组字过程就开始了。此时本地的IME软件（比如微软拼音输入法）会为我们提供**组字框**和**候选列表**的UI组件，如下图。

![ime](ime.png)
<center><div style="font-size:16px;">输入法组件</div></center>

与此同时，浏览器会收到操作系统文本服务（比如Windows的TSF框架<sup>[2]</sup>）的消息，并触发组字相关的事件，第一个事件就是`compositionstart`<sup>[3]</sup>。该事件表明组字阶段的开始，我们在Web应用程序中可以通过此事件识别用户是否开始使用输入法了。

## 参考阅读
[1] [Input Method Editor](https://en.wikipedia.org/wiki/Input_method)
[2] [Text Service Framework](https://docs.microsoft.com/en-us/windows/win32/tsf/text-services-framework)
[3] [Composition Start Event](https://developer.mozilla.org/en-US/docs/Web/API/Element/compositionstart_event)