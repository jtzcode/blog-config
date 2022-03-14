---
title: Web 键盘输入法应用开发指南——模拟事件
date: 2022-03-10 17:29:56
categories:
    - 技术
tags: 
    - 技术
    - 输入法
    - Web
    - JavaScript
---
## 引言
在这篇文章中，我们来来聊聊如何在Web应用中模拟各类事件的触发。有时候我们需要通过程序脚本，主动触发一些内置事件（如键盘、鼠标、触碰等），或者自定义事件，以满足业务需求。另外，在做Web程序的**自动化**时，模拟事件的触发也是必备的技能。

## isTrusted属性
在讨论模拟事件之前，我们有必要理解事件对象上的一个属性`isTrusted`<sup>[1]</sup>，它表示事件是否受信。一般来说，由**用户操作**产生的事件是受信的，该属性为`true`；而由**程序脚本**产生和触发的事件是不受信的，该属性为`false`。

浏览器**只会实现isTrusted值为true的事件的效果**。比如，你用脚本生成了一个keydown事件，并通过`EventTarget.dispatchEvent`把它派发给某个输入框控件元素。此时，如果给改控件添加事件处理程序，keydown事件的确会触发，不过`isTrusted`值一定为`false`，且**输入框内不会产生任何内容**。这里所谓事件的效果，就是UI控件上的变化（字符输入），以及后续事件

浏览器这样处理是出于**安全性**的考虑，试想如果通过脚本可以模拟真实输入事件，那么恶意脚本就可以修改页面的内容，模拟用户与页面的交互行为了，这是比较危险的。isTrusted属性是**只读**的，通过脚本无法改变其值，因此是安全的。

如果是这样的话，模拟事件是不是毫无意义呢？也不是。毕竟，你用脚本生成的事件对象，也可以指派给某个控件，并在适当时候触发，也可以通过事件处理器监听并采取相应行为。这可以帮助我们测试一些**键盘或者输入法场景的业务逻辑**，虽然并不能做UI上的改动。下面我们来看如何模拟事件触发。

## 模拟事件
由于我们这个系列以键盘和输入法应用为主题，这里就着重介绍键盘事件模拟，其他交互事件如鼠标、触屏可以参考公开的文档。定义一个键盘事件可以使用`KeyboardEvent`接口<sup>[2]</sup>:
```javascript
let keyEvent = new KeyboardEvent("keydown", {
    key: 'a',
    code: 'KeyA',
    location: 1,
    shiftKey: true,
    keyCode: 65,
    bubbles: true,
    cancelable: true
});

element.dispatchEvent(keyEvent);
```
可以直接给`KeyboardEvent`构造器传入**事件的类型**以及**初始化的选项**。有的程序会使用`initKeyboardEvent`方法来初始化事件对象，不过这样的写法已经被废弃<sup>[3]</sup>。最后调用`dispatchEvent`由相应的控件派发这个事件<sup>[4]</sup>，就可以使用event handler处理这个事件了。

事件的初始化选项支持很多字段，比如`key`，`code`，`bubbles`等，可以根据需要设置，具体支持列表可以参考文档<sup>[2]</sup>。你也可以在这里设置`isTrusted`属性，看看是什么效果。

同理，你也可以创建一个Input事件<sup>[5]</sup>：
```javascript
let inputEvent = new InputEvent("input", {
    bubbles: true,
    composed: true,
    data: "a",
    inputType: "insertText",
    isComposing: false
});
```
input事件一个关键之处就是它的`inputType`属性，在之前的文章中也介绍过，不同的值意味着截然不同的input事件行为。模拟事件的一般实践是，先通过调试器获取正常输入时的事件序列，记录事件的顺序和属性值，然后依次模拟生成。你甚至可以写一个程序来录制这个输入过程，自动生成模拟事件的代码。

## chrome插件

## 总结
## 参考阅读
[1] [isTrusted Property](https://developer.mozilla.org/en-US/docs/Web/API/Event/isTrusted)
[2] [Keyboard Event](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/KeyboardEvent)
[3] [initKeyboardEvent Method](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/initKeyboardEvent)
[4] [dispatchEvent Method](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/dispatchEvent)
[5] [Input Event](https://developer.mozilla.org/en-US/docs/Web/API/InputEvent)