---
title: Web 键盘输入法应用开发指南——输入事件
date: 2022-01-07 13:27:30
categories:
    - 技术
tags: 
    - 技术
    - 输入法
    - Web
    - JavaScript
---
## 介绍
在上一篇文章中，我简单介绍了与Web键盘和输入法应用相关的关键技术点。从这篇文章开始我们针对各个主题深入，看有哪些在开发中值得注意的地方。这里先从浏览器支持的与输入相关事件（按键、输入等）开始。

## keydown/keyup 事件
keydown<sup>[1]</sup>和keyup事件通常会成对出现（也有例外），它们**基本上**是用户在按下（释放）某个键时，浏览器最先触发的事件。这里还需注意两点：

1. 无论用户是否在使用输入法，keydown事件总是会触发，即便输入控件中没有任何字符。比如正在用输入法拼字，并且还没有提交。
2. 在一些特殊的平台或者浏览器（比如Android Chrome或者Mac Firefox等等），keydown事件不一定时最先触发的，一些输入法相关的事件（如compostionstart）会更早触发。这对于有些依赖于事件顺序的业务逻辑会造成影响，而且由于平台相关，与之相关的bug也不易发现。

### keyCode属性
这三个属性可以用来识别哪个键被按下了。key和code属性是字符串，而keyCode是一个整数。在一些比较老的应用上，我们会看到keyCode的广泛应用。比如keyCode=65表示字母a被按下，keyCode=18是Alt键，而keyCode=32则表示空格键。不过官方文档<sup>[2]</sup>已经不推荐使用这个属性了，因为它已经被Web标准逐渐废弃，各个浏览器厂商也会逐渐不再支持。keyCode值是因平台和浏览器而异的，在使用时要特别注意，具体的取值可以参考MDN的官方文档。

对于keyCode还有一些特殊情况，比如在使用输入法时，按下某个键会对应一个键盘上没有的字符（比如韩文的），这次按键其实也有keyCode，其取值通常较大（例如韩文字符基本在4000以上，准确的说是`keypress`事件的keyCode），这个信息可以作为判断当前输入语言的一个参考。对于中文和日文来说又不太一样，在拼写中文字符时，每次按键都会返回一个`229`的keyCode值，通过这个值可以过滤掉使用输入法时的keydown事件。

总的来说，使用keyCode要时刻考虑浏览器、语言及平台的兼容性，并且keyCode的值不易记忆，给开发和调试带来难度，因此对于新的应用不推荐使用。那么怎么拿到准确的键值呢？答案是**使用key和code属性**。

### key和code属性


### keypress事件

### 组合键相关属性

## 参考阅读

- [1] [Keydown Event](https://developer.mozilla.org/en-US/docs/Web/API/Document/keydown_event)
- [2] [KeyCode Property](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/keyCode)
