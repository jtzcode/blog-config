---
title: Web 键盘输入法应用开发指南——其他主题
date: 2022-03-17 15:34:11
categories:
    - 技术
tags: 
    - 技术
    - 输入法
    - Web
    - JavaScript
---
## 引言
这篇文章中，我们来看两个特别的话题。一个是关于特殊的`keyCode`属性<sup>[1]</sup>，另一个是关于键盘布局API的支持。虽然`keyCode`属性已被标准废弃，且不推荐使用，在一些老旧系统中，这个属性依然在返回作用，有些特殊情况还是要特殊处理。关于**键盘布局（Keyboard Layout）**，在这个系列的第一篇文章中就简单提及。因为这是一个涉及用户本地设备的的概念，浏览器迟迟没有提供较好的支持，后面会详细说明。

## 神秘的 keyCode 229
有的时候，我们会遇到`keyCode`属性值总是为`229`的情况。一般有两个原因：一是在**输入法启用**的状态，每个按键都用于输入法的组字过程，此时keyCode为229；二是在**安卓系统**上使用Chrome浏览器，当使用软键盘输入时，按键的`keyCode`始终为229<sup>[2]</sup>。其实`keyCode`的值应该设置什么时有章可循的，比如可以参考W3C的文档<sup>[3]</sup>。这里面有几个关键的点：

> 从操作系统的事件的虚拟键值（virtual key code）<sup>[4]</sup>读取

操作系统一般都有虚拟键的定义，比如在Windows上，针对某个应用程序的窗口，你可以使用类似Spy++的软件，抓取发送给这个窗口的按键事件，其中就有以`VK_`开头的属性，那就是虚拟键值。例如，虚拟键`VK_MENU`代表**ALT**键，其值为`0x12`。

虚拟键来自操作系统对键盘输入的处理，但是浏览器不一定能得到全部的信息，这个跟平台的实现相关。在Windows等系统上，键盘的事件还可能被输入法等软件“吃掉”，而无法到达浏览器端。因此，只凭借虚拟键设置`keyCode`不可靠。

> 如果是输入法处理按键，且类型是keydown，则返回229

因此对于输入法启用时，`keyCode`值为229是标准规定的。但不同的平台和浏览器出于各种考虑，不一定有相同的实现。对于安卓系统的情况就是一个例子，在使用软键盘时，安卓可能认为就是一种输入法的特殊情况，因此没有向浏览器发送全部按键信息，而总是返回229。况且真正输入的字符也可以通过其他事件获取到，不强依赖于`keyCode`属性。关于这种实现是不是一个bug有了很多讨论，感兴趣的话你可以查看这个帖子<sup>[5]</sup>。

注意，这里针对的是keydown和keyup事件，如果是keypress，则另有规则。keypress事件的`keyCode`经常被设置成对应字符的**Uncode码点**（code point），这在移动端经常用到，在PC端通过composition事件拿到的输入数据，在移动端可能要依赖keypress。

> 如果是数字和字母键，那么`keyCode`就是对应的**大写字母**的ASCII码

这就是为什么字母A的`keyCode`为65，而不是97，keyCode并不包含修饰键的信息，比如结合SHIFT、CapsLock等。接下来还有一些设置的规则，这里不一一列举，如果到最后没有合适的keyCode，那么就**设置为0**。

## 键盘布局API
## 总结
## 参考阅读
[1] [keyCode property](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/keyCode)
[2] [Android KeyCode 229](https://stackoverflow.com/questions/36753548/keycode-on-android-is-always-229)
[3] [keyCode Spec](https://lists.w3.org/Archives/Public/www-dom/2010JulSep/att-0182/keyCode-spec.html)
[4] [Windows Virtual Key](https://docs.microsoft.com/en-us/windows/win32/inputdev/virtual-key-codes)
[5] [Discussions on Android Chrome keyCode](https://bugs.chromium.org/p/chromium/issues/detail?id=118639)