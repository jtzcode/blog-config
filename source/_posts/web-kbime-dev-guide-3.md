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

另外，在一些带口音（accent）的欧洲语言中，有时候也需要输入法去拼出完整的口音字符，因为如果使用的是英文键盘，可能没有口音符号键来充当Dead Key，因此需要一个组字过程完成输入。

需要注意的一点是，如果仅仅是在标准的HTML可编辑控件中完成各类语言的输入，各种平台的浏览器已经做的很好了，不需要开发人员做过多的工作。只有当你的**Web应用程序需要干预用户的输入过程和结果**时，才需要关注具体的键盘和输入法事件。比如你想做一款在线的编辑器、输入法或者将输入定向到远端等功能特性，这些事件尤其有用。

## 输入法事件
`compositionstart`、`compositionupdate`和`compositionend`是一组事件<sup>[4]</sup>。首先是start被触发，组字框和候选列表相应出现；此后，每按一个新键，就会触发update，此时组字框和候选列表的内容也发生变化；当选择了候选列表中的某个字或词，或者敲击空格（中文输入法），end事件会触发，表明输入被提交。

在一些语言的输入法中有特殊的功能键。比如日文输入法，在开始输入后需要敲击空格键调出候选列表，此时的空格键并不是正常的空格键（keyCode为32，key为“ ”），而是被浏览器解释为**Process Key**（Windows上keyCode为**229**， key为“Process”）。实际上，用于提交输入的空格和回车键、从候选列表中选择目标字符的上下箭头键，以及调整光标位置的左右箭头键，也都是一种Process Key。这样应用程序就可以区分正常的按键和在使用输入法过程中的功能按键。

在输入过程的任何时候如果输入回车键，就会提交输入，`compositionend`事件也会触发。对于日文输入法，就是提交当前选中的字符；而对于中文输入法，则会提交组字框中的拼音。在使用输入法的过程中，也可以取消输入，一般有两种方式：一是用户的操作，比如使用鼠标单击页面空白处，就会终止当前输入；二是在`compositionstart`之后，通过`preventDefault`方法阻止后续事件发生<sup>[5]</sup>。但无论何种情况，`compositionend`事件都会触发。

通常情况下，`compositionstart`会开启一个组字的会话（session），然后会有一个或者多个`compositionupdate`事件描述输入的过程，最后随着`compositionend`事件提交输入结果，结束组字会话。在这期间，每次`compositionupdate`事件触发，都意味着DOM马上要更新为最新的字符，DOM更新结束后还会紧跟着一个`input`事件，表示当前字符更新成功。不过还是那句话，此为标准中定义的事件序列，并不是所有浏览器都如此实现。

在一个会话中，每一次非打印字符的输入都被视为`Process`键。比如在日文输入法中使用空格来调出候选列表，并选择候选字词；在中文输入法中使用回车来提交当前输入；在韩文输入法中使用Hanja键来切换到汉字输入等等。这些Process键的keydown或者keyup事件中，`isComposing`属性的值都是`true`，表示输入未结束，这对应用程序来说可能是个有用的属性。随着会话结束， 在compositionend事件触发以后，从最后一个Process键（比如回车键）的keyup事件开始，`isComposing`就变为`false`。

值得一提的是，在某些输入法中有`selection`的概念，即输入过程支持**分段组字**。如下图，这在拼写的是“地”这个字符，候选列表里有多个选项。此时使用左右方向键可以切换拼写的目标（即一个selection），比如切换到右边的“安气”，具体几个字一组与输入的语言相关，由输入法自行决定。同时，在浏览器中会有`compositionupdate`事件产生。

![japanese](ja.png)
<center><div style="font-size:16px;">日语输入法分段组字</div></center>
## input事件
## 总结

## 参考阅读
[1] [Input Method Editor](https://en.wikipedia.org/wiki/Input_method)
[2] [Text Service Framework](https://docs.microsoft.com/en-us/windows/win32/tsf/text-services-framework)
[3] [Composition Start Event](https://developer.mozilla.org/en-US/docs/Web/API/Element/compositionstart_event)
[4] [Composition Events](https://w3c.github.io/uievents/#events-compositionevents)
[5] [Cancel Composition](https://w3c.github.io/uievents/#events-composition-canceling)