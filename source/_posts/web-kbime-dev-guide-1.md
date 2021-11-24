---
title: Web 键盘输入法应用开发指南（一）
date: 2021-10-15 16:18:37
categories:
    - 技术
tags: 
    - 技术
    - 输入法
    - Web
    - JavaScript
---

## 介绍
在Web应用中，用户交互是十分基本和重要的功能，而来自用户的输入又离不开键盘。现代Web浏览器已经在底层对键盘做了**相对**较好的支持（逐渐就能看出为什么是“相对”），并通过API的方式暴露给开发者。这包括针对按键的事件捕获，不同语言键盘布局（Keyboard Layout）的支持，对输入法的支持（涉及中、日、韩等东亚语言），以及对不同平台（Windows、Mac、iOS等）上组合键的支持，等等。在这个系列的开头，我们来过一遍浏览器提供的这些与键盘输入相关的基本技术特性，先有一个基本印象，后续再逐渐展开。

## 按键事件
对于按键事件（Keyboard Event）[1]大家应该都不陌生，在开发Web应用时或多或少应该接触过。一般来说，这类事件是你在键盘上按下某个键时**最先触发**的事件。“最先”的意思是还有后续的事件类型产生（比如`input`事件等），用于进一步的输入逻辑处理。而到这一步，你（或者说Web应用)能知道的是，**浏览器从操作系统那里得到的按键信息**。至于说这个按键会得到怎样的输入和文本，那就是后续事件的事情了。我前面还加上了“**一般来说**”，这是因为不同的浏览器，针对不同的输入场景（是否使用输入法），还有不同的实现。

浏览器将按键事件分为三类：keydown、keyup 和 keypress，通过Keyboard Event对象的`type`属性来区分。前两类的含义可以从字面理解，而keypress类型一般是针对**产生具体字符**的按键（比如单按一个Ctrl键就不会产生字符），但已不推荐使用了[2]。有了这些类型，就可以在应用中对用户的键盘输入做比较精细的处理。一个简单的例子是，我要知道用户按住了某个特定的键，持续一段时间，以执行某个业务逻辑，并在释放按键时终止逻辑。

按键事件对象里还有一些常用的属性，比如`code`属性告诉你哪个键被按了，`location`属性告诉你这个键在键盘的哪个区域（是左Alt还是右Alt），`isCompsing`属性告诉你这个键是不是在使用输入法期间按下的，等等。当然，这些属性很多都有兼容性问题，我会在后面细说。

## 键盘布局
简单来说，键盘布局[3]（Keyboard Layout）规定了你的键盘上某个位置的键将会产生怎样的输出，这是一个映射关系。比如在Windows系统上，我们输入中文和英文，其实都在使用英文键盘布局，而输入日文和德文就需要相应的日文和德文键盘布局了。在英文布局上，你按下“y”这个键，结果就输出“y”这个字符，而在德文布局下，同样的键会产生“z”字符。键盘布局在你切换输入语言或者输入法时会进行相应变换，这些由系统完成。

![kbime-switch](keyboard-ime-switch.png)
<center><div style="font-size:16px;">系统键盘/输入法切换</div></center>

在一些系统上有`Scan Code`的概念，一个键的scan code标识了这个键在当前所使用键盘上的**物理位置**。比如在常用的英文键盘上，左边shift键旁边的位置就是字母“z“，那么它的scan code就标识了这个键的位置在左shift的右边，仅此而已。至于按下键后会输出什么字符，这就需要键盘布局的信息了。系统会根据当前的键盘布局，将来自物理键盘的scan code信息“翻译”成应该输出的字符。

![de-keyboard](de-keyboard.png)
<center><div style="font-size:16px;">德语键盘布局</div></center>

现代浏览器的实现中没有直接拿到scan code，但通过键盘事件中的`code`属性[4]，可以访问到类似的位置信息。例如按下字母“Q”键产生的keydown事件中`code`属性的值为`KeyQ`，它的含义是英文键盘上**对应Q的位置的键被按下了**。注意IE浏览器不支持code属性，在开发时要考虑用户的场景。在这里[5]可以找到更详细的针对不通过键盘的code定义。

那么浏览器能知道用户当前的系统键盘布局吗？答案是：**目前还不能**。因为这需要浏览器通过系统API得到需要的信息，而浏览器还没有提供JavaScript接口。目前只有个`getLayoutMap`接口[6]，且还没有被所有主流浏览器支持。它返回一个promise，并在promise被resolve时拿到一个**从按键信息到字符信息的map**。这样只要通过事件对象拿到按键信息（比如`code`属性），就可以得到它在当前系统键盘布局下的输出字符，从而可以**反推**目前系统的键盘布局信息。当然，如此获得键盘布局信息还是略显笨拙，还是期待未来标准中对键盘信息有更为完善的支持。

## 输入法事件
## 组合键
## 总结

## 参考资料
- [1] Keyboard Event：https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent
- [2] Keypress Event: https://developer.mozilla.org/en-US/docs/Web/API/Document/keypress_event
- [3] Keyboard Layout: http://taggedwiki.zubiaga.org/new_content/b996676d2852e3818a60c2655ac10e73#:~:text=A%20keyboard%20layout%20is%20any%20specific%20mechanical%2C%20visual%2C,layout%3A%20The%20placements%20and%20keys%20of%20a%20keyboard
- [4] Code property: https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/code
- [5] Code values: https://www.w3.org/TR/uievents-code/
- [6] Keyboard Layout Map: https://developer.mozilla.org/en-US/docs/Web/API/Keyboard/getLayoutMap