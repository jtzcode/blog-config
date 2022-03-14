---
title: Web 键盘输入法应用开发指南——实战（二）
date: 2022-03-09 09:56:58
categories:
    - 技术
tags: 
    - 技术
    - 输入法
    - Web
    - JavaScript
---
## 引言
在这篇文章中，我们来完成在线输入法（Online IME)小程序的基本功能。实现的点包括，使用SHIFT键来切换中英文输入状态，给候选列表提供分页的功能，并且在适当的时候阻止composition事件的处理。跟商业输入法相比，这里实现的功能还是极为有限，但这两篇实践可以给你一个实现层面的基本印象。

## 新增功能
### 使用shift键切换输入语言
实现这个功能本身并不难，只需要注意一些容易出错的点。我们默认使用的是中文输入法，即激活组字过程，通过服务端API返回实时组字结果，展示候选词列表。

<!--more-->

```javascript
if (!evt.key.match(digitsReg) && isLetter(evt.key) && isIMEEnable) {
    textBuffer += evt.key;
    startComposition(textBuffer);
    evt.preventDefault();
    evt.stopPropagation();
}
```
使用变量`isIMEEnable`控制这个SHIFT的状态，通过`startComposition`开始组字，并**阻止事件的默认行为**。按过以此SHIFT键后，需要停止这个过程，直接使用按键事件的默认行为：

```javascript
inputArea.addEventListener('keyup', evt => {
    if (evt.key === 'Shift' && !evt.altKey && !evt.ctrlKey &&           lastKeydown === 'Shift') {
        isIMEEnable = !isIMEEnable;
        lastKeydown = null;
    }
});
```
可以在`keyup`事件中查看SHIFT键的状态，然后更新`isIMEEnable`的值。注意，由于我们需要的是单击SHIFT键的功能，而不是**组合键中SHIFT**的功能，因此要保证其他功能键（ALT、CTRL等没有同时按下）。比如针对ALT+SHIFT组合，我们可以不做处理：

```javascript
// keydown event
if (evt.key === 'Shift' && (evt.altKey || evt.ctrlKey)) {
    return true;
}
```
最后还有一个小问题，就是用户可能按的是SHIFT+A之类的组合键，输出大写字符，并且**两个键的释放顺序不一定**。因此在遇到SHIFT的keyup事件时，要**判断上一个keydown事件是什么**，如果也是SHIFT，说明没有组合键的情况。

在上面的代码中，使用了`lastKeydown`来做这个判断。像这类特殊处理，在实际的开发中经常遇到，要多测试，包括功能和回归，总有事先想不到的情况。

### 候选列表分页
前面的实现中，在每次从服务器请求候选词列表时，我们指定了固定的数量，注意服务端实际请求URL的参数：

`/request?itc=zh-t-i0-pinyin&text=a&num=10&cp=0&cs=1&ie=utf-8&oe=utf-8&app=demopage`

这里的`num`参数可以指定返回的列表大小。要实现分页，首先要动态传递这个参数，开始时传入默认值（比如10），然后用户在翻下一页时（比如按了**向下箭头键**），再请求新的候选列表。

```javascript
...
else if (evt.code === 'ArrowDown') {
    getMoreCandidates();
}
...
function getMoreCandidates() {
    const url = `/more?text=${textBuffer}&num=${pageNum * candidateWindowSize}`;
    fetch(url).then(res => {
        ...
    });
}
```
`pageNum`默认为1，后续可以修改它的值，并通过`pageNum * candidateWindowSize`控制整个候选词列表的大小。

当然，我们也可以稍作优化，不在每次用户按箭头键时请求更多的候选项，而是预先多取一定数量的的词，然后当翻页不够显示时，再请求新的项。这样可以**减少网络请求的次数**，优化性能。

### 阻止composition

一般来说，这个在线输入法应用不需要处理composition事件，因为我们在keydown事件中阻止了事件的默认行为，即不会触发后续的composition相关事件。然而，正如在前面系列文章中提到的，在一些平台（系统+浏览器）组合上，**composition是有可能先于keydown事件**触发的。因此对于composition事件，我们最好把它也阻止掉。

```javascript
inputArea.addEventListener('compositionstart', evt => {
    evt.preventDefault();
    return false;
});
```
对`compositionstart`事件调用`preventDefault`，是取消输入过程的一般做法。

## 未实现功能
到这里，这个简单的在线输入法就完成了，勉强算麻雀虽小五脏俱全吧。感兴趣的话你可以进一步的优化。这里有几个建议。
1. 候选列表优化。候选列表目前只支持使用数字键选择，未来可以支持箭头键切换选项，以及鼠标选择。并且，目前所有候选词只是一个字符串，没有使用UI上的元素。
2. 通用性插件。相信你也注意到了，这个输入法目前只是在特定的文本框输入，你可以去支持更通用的输入框支持，将文本框的信息作为参数传递。或者更加通用一些，将输入法做成浏览器插件[2]，在任意时刻、任意的可输入控件中，都可以开启输入法插件。
3. 更多语言。对于其他语言输入法的支持，思路是类似的，不过在输入过程以及UI的设计上可能会更复杂。

## 总结
我们利用两篇文章实现了一个简单的在线输入法，完整的代码可以参考资料[1]。当然，这个简单的应用并没有涉及所有与键盘和输入法相关的主题，不过你可以发挥想象力，实现更多的功能，体会这类应用开发的特点。下篇文章我会介绍如何在浏览器中**模拟事件**。

## 参考阅读
[1] [Online-IME Demo](https://github.com/jtzcode/online-ime)
[2] [Goole Input Tools Extension](https://www.google.com/inputtools/try/)