---
title: Web 键盘输入法应用开发指南——键盘事件
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

## keydown/keyup/keypress 事件
keydown<sup>[1]</sup>和keyup事件通常会成对出现（也有例外），它们**基本上**是用户在按下（释放）某个键时，浏览器最先触发的事件。这里还需注意两点：

1. 无论用户是否在使用输入法，keydown事件总是会触发，即便输入控件中没有任何字符。比如正在用输入法拼字，并且还没有提交。
2. 在一些特殊的平台或者浏览器（比如Android Chrome或者Mac Firefox等等），keydown事件不一定时最先触发的，一些输入法相关的事件（如compostionstart）会更早触发。这对于有些依赖于事件顺序的业务逻辑会造成影响，而且由于平台相关，与之相关的bug也不易发现。

keypress事件<sup>[7]</sup>已经被标准废弃，不建议继续使用。它表示的是实际产生字符值的按键事件，比如ALT、CTRL之类功能键就不会触发该事件。不过即使产生了字符输出，该事件也不一定触发，这跟浏览器的实现有关，关键逻辑建议不要依赖这个事件。

## keyCode属性
在一个按键事件中，key、code和keyCode这三个属性可以用来识别哪个键被按下了。key和code属性是字符串，而keyCode是一个整数。在一些比较老的应用上，我们会看到keyCode的广泛应用。比如keyCode=65表示字母a被按下，keyCode=18是Alt键，而keyCode=32则表示空格键。不过官方文档<sup>[2]</sup>已经不推荐使用这个属性了，因为它已经被Web标准逐渐废弃，各个浏览器厂商也会逐渐不再支持。keyCode值是因平台和浏览器而异的，在使用时要特别注意，具体的取值可以参考MDN的官方文档。

对于keyCode还有一些特殊情况，比如在使用输入法时，按下某个键会对应一个键盘上没有的字符（比如韩文的），这次按键其实也有keyCode，其取值通常较大（例如韩文字符基本在4000以上，准确的说是`keypress`事件的keyCode），这个信息可以作为判断当前输入语言的一个参考。对于中文和日文来说又不太一样，在拼写中文字符时，每次按键都会返回一个`229`的keyCode值，通过这个值可以过滤掉使用输入法时的keydown事件。

总的来说，使用keyCode要时刻考虑浏览器、语言及平台的兼容性，并且keyCode的值不易记忆，给开发和调试带来难度，因此对于新的应用不推荐使用。那么怎么拿到准确的键值呢？答案是**使用key和code属性**。

## key属性
简单来说，key属性<sup>[3]</sup>告诉你的按键产生了什么**值**，这个属性名稍有些迷惑性（叫“键”，实际是“值”）。比如在一个英文键盘上，你按“A”键，那对应的key event中key属性的值就是“a”。

需要注意的是key的值是**最终的**输出值，也就是它也考虑了SHIFT等功能键的效果、系统的locale以及键盘的layout等信息。比如，如果按的是**SHIFT+A**组合，那么key的属性值是大写的“**A**"；在韩文输入法下，按右边的CTRL键，其实是韩语输入的Hanja key（用于输入汉字），对应key的属性值为“**HanjaMode**”（而不是CTRL本身）；在日语的106/109键盘上，按特殊的Kana/罗马字转换键，会产生“**KanaMode**”的key属性值。

注意，产生的这个key值如果是一个可打印的字符（比如字母、数字、符号等），那么会进一步出发`input`等事件，将字符输入某个文本控件内。如果只是功能键（比如Hanja key）则不会有`input`事件产生。因此，有时我们在拦截用户输入时`input`事件不一定来得及，并不是每次按键都产生实际输入。

在一些输入法的场景中，有用的key属性可能出现在**keypress**事件中。比如使用中文输入法输入一些全角字符时（中文的`￥`，`【`，`》`等等），在keydown和keyup事件中，key属性可能只是“**Process**”，表示这个按键被输入法处理了，而在keypress事件中的key属性却可以得到准确的中文全角字符。针对这种情况，开发者还要注意平台的差异性，根据我的经验，在**Ubuntu的Firefox**浏览器中，keypress事件就无法产生，可以认为是一个bug。不过在上诉场景中，通常还有`input`事件产生，可以用来输入的处理，但不一定是好的时机。

如果所在的平台和浏览器无法识别某个键盘的按键，那么按键事件依然会产生，只不过key属性的值为“**Unidentified**”。在处理按键事件时要时刻牢记三个影响因素：**键盘布局、操作系统和浏览器**。还有一种特殊的key称为Dead Key<sup>[5]</sup>，在输入某些欧洲语言的accent字符时，需要先按一次口音符号\`，再按字母键（比如在法语中的**à=\`+a**），这里的口音符号就会产生“**Dead**”的key值，而a键的key属性为最终的字符`à`。

## code属性
有时候，我们可能不关心某个键的输出是什么，而只是关心当前键盘上哪个位置的键被按下了，这其实是像获取按键的`Scan Code`。Scan Code在不同的平台上有不同的表示，同时还与外接的键盘有关（比如英语的101键盘，日语的106键盘，韩语的103键盘等等）。对应到浏览器平台，并没有scan code的概念，而是通过code属性<sup>[4]</sup>表达类似的功能。

按照W3C组织的标准<sup>[6]</sup>定义，code属性包含了用户所按**物理键**的信息。既然是物理键，那么就跟键盘的类型有关系，比如下图就是一个典型的英文101-键盘的布局。

![kbime-101-layout](keyboard-101-us.svg)
<center><div style="font-size:16px;">英文101键盘</div></center>

这里还有一个典型的日语的106键盘布局，可以看到其中多了一些日语输入相关的功能键。

![kbime-106-layout](keyboard-106-japanese.svg)
<center><div style="font-size:16px;">日文106键盘</div></center>

无论是什么样的键盘布局，上面键的类型总可以分成那么几类。以字母数字（Alphanumeric）区域为例，常见的字母和数字键就可以称为**书写系统（Writing System）键**，而像ALT、SHIFT和KanaMode这样的就是**功能（Functional）键**。但无论哪种类型的键，都有一个唯一的名称或者ID，这个就是code属性的值。比如下图中的**“KeyA”就是字母A键产生的按键事件的code属性值**，不管你使用的是什么键盘布局，这个code值是不会变的，因为它表示的是键的位置，而不是产生什么输出。

![kbime-keycodes](keyboard-codes-alphanum1.svg)
<center><div style="font-size:16px;">Key Code</div></center>

另外需要注意的是，图中紫色的键是**任何**键盘布局都有的键，而绿色的键是只有**特定**布局的键盘才有。比如回车键**左边**的“BackSlash”键，在英文键盘上就不存在。通过这个code属性，我们不仅能知道哪个键被按下了，还能了解一些**客户端目前键盘布局**的信息。比如你收到了一个值为“KanaMode”的code属性，那使用的键盘布局一定是日语键盘，因为其他类型的键盘不可能产生这个属性值。

## 其他属性
按键事件中还有一些常用的属性，比如与组合键相关的`altKey`、`ctrlKey`和`shiftKey`，它们表示某个键被按下时，是否同时按住了某个功能键。这在处理一些快捷键相关逻辑时比较有用。例如处理ALT+F的组合，可以监听keydown事件，如果是`F`键被按下，同时检查它的altKey属性，如果为`true`，该组合键就生效了。

location属性表示按键所在的区域，比如对于ALT键来说，如果是左边的ALT，该值为`1`；如果是右边，则为`2`。如果涉及小键盘区的键，取值为`3`也有可能。有些情况下这个属性也有用，比如Windows韩语输入法中，右边的CTRL键是作为`HanjaMode` key使用的，除了通过code属性区分，location也可以。而在某些欧洲语言键盘布局中，右边的ALT键是`AltGraph`键，在判断keyCode的同时，通过location也可以与左边的ALT区分开来。

repeat表示某个键是否被一直按着，当长按某个键时触发。按多久算长按，每个系统平台可能有所区别，只有平台认为长按触发，该属性才是`true`。

## 总结
本文介绍了浏览器中针对键盘事件的知识和注意事项，每个事件和属性的具体表现还要根据具体开发实践而定，标准写了不代表每个平台和浏览器都正确实现了，在开发时要勤查文档，但也不能迷信标准。从下篇文章开始，我们来看看输入法相关的事件。

## 参考阅读

- [1] [Keydown Event](https://developer.mozilla.org/en-US/docs/Web/API/Document/keydown_event)
- [2] [KeyCode Property](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/keyCode)
- [3] [Key property](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/key)
- [4] [Code property](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/code)
- [5] [Dead Key](https://en.wikipedia.org/wiki/Dead_key#/)
- [6] [UI-Event Code](https://www.w3.org/TR/uievents-code/)
- [7] [Keypress Event](https://developer.mozilla.org/en-US/docs/Web/API/Document/keypress_event)
