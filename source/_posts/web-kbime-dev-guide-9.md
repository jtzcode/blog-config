---
title: Web 键盘输入法应用开发指南——标准与实现
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
这篇文章中，我们通过两个特别的主题来探讨标准与实现的问题。在Web前端领域，虽然标准总是先行，但浏览器厂商以及各平台是否跟进，却有着自己的考量。在前面的文章中，我们已经多次提到过具体的案例，本文中案例一个是关于特殊的`keyCode`属性<sup>[1]</sup>，另一个是关于键盘布局API的支持。

虽然`keyCode`属性已被标准废弃，且不推荐使用，在一些老旧系统中，这个属性依然在发挥作用，有些特殊情况还是要处理。关于**键盘布局（Keyboard Layout）**，在这个系列的第一篇文章中就简单提及。因为这是一个涉及用户本地设备的**数据隐私**的API，浏览器迟迟没有提供较好的支持，后面会详细说明。
<!--more-->
## 神秘的 keyCode 229
有的时候，我们会遇到`keyCode`属性值总是为`229`的情况。一般有两个原因：一是在**输入法启用**的状态，每个按键都用于输入法的组字过程，此时keyCode为229；二是在**安卓系统**上使用Chrome浏览器，当使用软键盘输入时，按键的`keyCode`始终为229<sup>[2]</sup>。其实`keyCode`的值应该设置什么时有章可循的，比如可以参考W3C的标准文档<sup>[3]</sup>。这里面有几个关键的点：

> 从操作系统的事件的虚拟键值（virtual key code）<sup>[4]</sup>读取

操作系统一般都有虚拟键的定义，比如在Windows上，针对某个应用程序的窗口，你可以使用类似Spy++的软件，抓取发送给这个窗口的按键事件，其中就有以`VK_`开头的属性，那就是虚拟键值。例如，虚拟键`VK_MENU`代表**ALT**键，其值为`0x12`。

虚拟键来自操作系统对键盘输入的处理，但是浏览器不一定能得到全部的信息，这个跟平台的实现相关。在Windows等系统上，键盘的事件还可能被输入法等软件“吃掉”，而无法到达浏览器端。因此，只凭借虚拟键设置`keyCode`并不可靠。

> 如果是输入法处理按键，且类型是keydown，则返回229

因此对于输入法启用时，`keyCode`值为229是标准规定的。但不同的平台和浏览器出于各种考虑，不一定有相同的实现。对于安卓系统的情况就是一个例子，在使用软键盘时，安卓可能认为就是一种输入法的特殊情况，因此没有向浏览器发送全部按键信息，而总是返回229。况且真正输入的字符也可以通过其他事件获取到，不强依赖于`keyCode`属性。关于这种实现是不是一个bug有了很多讨论，感兴趣的话你可以查看这个帖子<sup>[5]</sup>。

注意，这里针对的是keydown和keyup事件，如果是keypress，则另有规则。keypress事件的`keyCode`经常被设置成对应字符的**Uncode码点**（code point），这在移动端经常用到，在PC端通过composition事件拿到的输入数据，在移动端可能要依赖keypress。

> 如果是数字和字母键，那么`keyCode`就是对应的**大写字母**的ASCII码

这就是为什么字母a的`keyCode`为65，而不是97，keyCode并不包含修饰键的信息，比如结合SHIFT、CapsLock等。接下来还有一些设置的规则，这里不一一列举，如果到最后没有合适的keyCode，那么就**设置为0**。

## 键盘布局API
关于键盘布局的API，我在第一篇文章中简单提及，那就是还未被广泛支持的`navigator.keyboard.getLayoutMap`方法<sup>[6]</sup>。它可以获得本地键盘布局的信息，通过Promise返回。这个方法未被广泛支持的原因是它在一定程度上违反了用户设备数据的**隐私性**（Privacy）。

关于隐私性我的理解是，浏览器不能在**用户没有感知**的情况下，**读取**本地设备上的**敏感信息**。这里有几个关键点，首先是用户没有感知，如果涉及了用户交互，比如征求用户同意，且用户可以选择接受或者拒绝，就不算违反隐私性。然后是**读取**这个操作，这是相对于修改、篡改而言的，后者就不仅仅是隐私性问题了，而是归为**安全性（Security）**问题。比如，我们上篇文章提到的禁止通过脚本修改设备上软键盘的信息，就涉及安全性考虑。

最后一点是"敏感信息"的定义。我们通常可以理解为与用户的痕迹、习惯、偏好相关的信息，但也不是太明确。你认为本地使用的键盘布局是敏感信息，别人可能觉得无所谓。不过目前看来，键盘布局的API迟迟没有被浏览器广泛支持，还是因为认可它的隐私性的。这里有个微软关于该API的需求的消息<sup>[7]</sup>。

这个API之所以有需求，是因为它在某些场景下带来了用户体验的提升。比如，应用程序可以根据当前的系统的键盘布局的变化，切换处理输入的逻辑，让实现更高效，更契合当前输入的语言和键盘布局的特点。由于目前开发者无法拿到这类信息，因此对多种语言和键盘的兼容性显得力不从心。下面我们通过代码来看看这个API的用法。

首先可以通过[caniuse](https://caniuse.com/?search=getlayoutmap)网站查看该API的兼容性，我直接使用最新版本的Chrome（如99）。其次，你的测试程序要通过**安全连接**访问（如HTTPS)。如果连接是非安全的，Chrome浏览器中该API依然是不可见的。因此，你可以使用本地文件系统打开HTML文件；如果通过HTTP访问，则需要使用localhost作为域名，不要使用IP，因为localhost也会被认为是基本安全的。

```javascript
...
function checkKeyboardLayout() {
    var keyboard = navigator.keyboard;
    keyboard.getLayoutMap().then(keyboardLayoutMap => {
        var targetLayout = charToLayouts.find((layout) => {
            var kv = keyboardLayoutMap.get(layout.key);
            return kv === layout.value;
        });
        if (targetLayout && targetLayout.result !== currentLayout) {
            currentLayout = targetLayout.result;
            console.log("Keyboard layout switched to: ", targetLayout.result);
        } else {
            console.log("Unsupported keyboard layout selected");
        }
                                                                
    });
}
...
```
任何时候，可以通过`navigator.keyboard.getLayoutMap()`获得一个**从字符code到key**的map（`keyboardLayoutMap`），即给定一个键位（code）的信息，这个map可以返回在当前键盘布局下，产生什么字符输出（key）。有了这个map，我们可以通过尝试一些比较特殊的键位，从而推断出键盘布局是什么。比如，我们事先定义了一个`charToLayouts`的表：

```javascript
const charToLayouts = [
    {
        key: 'KeyY',
        value: 'z',
        result: 'de-DE'
    },
    {
        key: 'Digit2',
        value: 'é',
        result: 'fr-FR'
    },
    ...
];
```
通过逐一尝试表中的每一项，将`key`字段传递给`keyboardLayoutMap`，看输出是否是`value`对应的字符。如果是，则推断出了当前的键盘布局。比如数字2键位的返回值是字符`é`，说明当前键盘布局是法语键盘。这种实现目前还有两个限制：
1. API返回的map不包含所有键的映射，目前只使用可打印字符（比如，字母、数字和符号等）
2. 没有相应的事件通知键盘布局的变化

关于第二点，其实相关标准<sup>[8]</sup>里面已经提供了`layoutchange`事件，只不过包括Chrome在内的所有浏览器都**没有实现**它。如果没有这个事件，我们应用程序不好决定何时进行上述的键盘布局检测。我个人觉得，这个API未来被全面支持的希望不大，可能会默认关闭，然后可以通过浏览器的某种隐私设置打开它。

## 总结
本文借两个案例探讨了标准与实现的问题。各个平台和浏览器厂商对某个API或者语言特性的支持，需要从**向前兼容性、平台特性、实现复杂性、隐私与安全性以及未来的可能性**等多个方面综合考虑，因此不一定完全按照标准来实现。这不仅体现在键盘和输入法相关的实现中，也是前端开发的常见问题。当遇到实现与标准不一致时，搞清楚为什么会这样，可能会给我们理解所使用的技术甚至业务带来启发。
## 参考阅读
[1] [keyCode property](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/keyCode)
[2] [Android KeyCode 229](https://stackoverflow.com/questions/36753548/keycode-on-android-is-always-229)
[3] [keyCode Spec](https://lists.w3.org/Archives/Public/www-dom/2010JulSep/att-0182/keyCode-spec.html)
[4] [Windows Virtual Key](https://docs.microsoft.com/en-us/windows/win32/inputdev/virtual-key-codes)
[5] [Discussions on Android Chrome keyCode](https://bugs.chromium.org/p/chromium/issues/detail?id=118639)
[6] [Keyboard Layout Map](https://developer.mozilla.org/en-US/docs/Web/API/Keyboard/getLayoutMap)
[7] [Microsoft'S requirement on getLayoutMap API](https://www.theregister.com/2022/01/06/chrome_97_privacy/)
[8] [Keyboard Map Spec](https://wicg.github.io/keyboard-map/)