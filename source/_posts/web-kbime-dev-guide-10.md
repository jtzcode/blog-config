---
title: Web 键盘输入法应用开发指南——性能与原理
date: 2022-03-21 10:14:31
categories:
    - 技术
tags: 
    - 技术
    - 输入法
    - Web
    - JavaScript
---
## 引言
在本文中，我们先来讨论事件处理时的**性能**问题，尤其是针对键盘和输入法事件处理流程的性能优化。然后我们稍微深入到浏览器的实现，探究一下从用户按下键盘，到输入的文本出现在页面上，中间经历了什么。

## 性能优化
Web应用程序的性能因素有很多，即使只考虑前端的因素也不少，如浏览器缓存、页面渲染、JavaScript的解析和执行等等。我们这里只关注与**UI事件处理**相关的性能点。

### JavaScript优化
首先考虑JavaScript本身的写法。通常来说，我们给一个输入控件添加事件处理程序（如keydown）后，这个处理程序会在短时间内频繁被调用，比如输入一段文字。如果是与鼠标拖动和滚动相关的事件，可能会更加频繁。此时，在处理程序内部的JavaScript的性能问题就值得关注了。尽管目前浏览器的渲染引擎和JS引擎性能都比较强大，对于复杂业务逻辑来说，注重代码的性能仍是十分有益的。这里结合《高性能JavaScript》一书<sup>[1]</sup>，给出几个建议。

<!--more-->

1. 标识符解析的性能
一般来说，标识符所在的位置越深，读写速度越慢。对于在循环中或者在频繁触发的事件处理器中使用的变量，最好将变量暂存为临时变量，避免多次无效读取。
```javascript
const keydownHandler = (evt) => {
    if (glob.environment.app.isMyApp) {
        ...
    }
}
```
这段变量访问完全可以提到事件处理之外，以减少调用。

2. 注意作用域的影响
有一些语法特性会临时**改变作用域链**，而这是由性能损耗的。比如try-catch块，当程序发生异常进入catch块时，这里定义的局部变量都会加入作用域链，放在异常对象之后，异常处理。因此好的做法是使用一个独立的函数处理异常，这样作用域链中就只有头部一个对象（ex）：
```javascript
try {
    thisWillThrowException();
} catch (ex) {
    HandleException(ex);
}
```
另外闭包也有类似的问题，比如在一个函数执行时绑定了一个事件处理函数，并访问了局部变量：
```javascript
function foo() {
    var id = "12";
    document.getElementById("input").onkeydown = (evt) => {
        handleKey(evt, id);
    };
}
```
keydown事件处理器访问了`id`变量，因此是一个闭包，它在每次`foo`函数调用时都会创建以便，不是一个好的做法。

3. 减少DOM的修改
有时我们需要根据键盘事件来修改DOM元素的行为，修改行为本身就是有代价的，**如果涉及了页面的重排、重绘的过程，则代价更高**。因此，尽量避免直接使用JavaScript修改DOM，而是改用CSS等方式。如果修改是必要的，也要避免多次重复操作（比如在循环或者事件处理器中）。

### 事件处理器数量
最好不要给过多的元素添加事件处理器。[实验表明](https://web.archive.org/web/20170121035049/http://jsperf.com/click-perf)，这种做法会显著降低页面性能，尽管每个元素只绑定一个事件处理程序<sup>[2]</sup>。这还只是绑定事件的操作，没有涉及具体的事件处理程序被调用的性能。你可以从这个基于jQuery的实验页面找到一些数据：

![perf](perf.png)
<center><div style="font-size:16px;">添加事件处理程序的性能</div></center>

通过`.many`样式添加的`click`事件处理程序会作用于大量元素，因此速度最慢。而通过父元素添加事件处理程序有着最好的性能，这就是**事件委托**（Event Delegation）。

### 事件委托
事件委托<sup>[3]</sup>就是为了解决上述问题，通过在父元素添加事件处理程序，避免给过多子元素添加。理想情况下，我们只需要给**document文档对象**添加需要的事件处理器即可：
```javascript
document.addEventListener('click', function(event) {
    let id = event.target.id;
    if (!id) return;
    let elem = document.getElementById(id);
    ...
});
```
然后通过event对象的`target`属性来识别到底是哪个子元素触发了该事件，进行相应的处理。不过我们一般不会在`document`上直接使用事件委托，而是在DOM树的某个较大节点上使用，用于处理其子节点的事件。这样可以避免比必要的事件处理。事件委托还有一个好处是，当删除一个子元素时，我们不必要考虑删除其对应的事件处理程序，而是由父元素统一管理了。

看起来这个做法不错，不过事件委托也有一些问题。比如，它要求**事件冒泡不能关闭**，要一直冒泡到目标父元素上，这个程序的实现带来了潜在的限制。另外一点涉及连续触发的事件，如鼠标滚轮的`wheel`事件，或者触屏事件`touchstart`。

浏览器在处理这类事件时，会涉及**合成线程（Compositor）和主线程的交互**。在现代浏览器架构中，页面渲染和脚本执行是有一个独立的**渲染进程**完成的。而渲染进程又创建了多个线程，比如处理网络和输入事件的**I/O线程**，页面渲染的**主线程**，以及负责合成图层的**合成线程**。

因为事件处理程序需要在主线程中执行，因此合成线程需要在合适的时候通知主线程调用事件处理程序。合成线程会把页面上**有事件处理程序的区域**标记为“**非快速滚动区域**”（Non-Fast Scrollable Region）。如果不在这类区域中，合成线程会直接合成下一帧，而不会等待主线程，从而优化性能<sup>[4]</sup>。

![render](nonfast-scroll.png)
<center><div style="font-size:16px;">非快速滚动区域</div></center>

在事件委托的场景下，可能整个页面都在监听连续事件，那么整个document都是非快速滚动区域，那么合成线程就会不停地**等待主线程的执行**结果。不过此时可以给事件处理程序传递`passive`属性进行优化，详细可以参考文档<sup>[5]</sup>。

## 深入事件处理
下面我们以Chrome为例深入浏览器内部探究一下，从用户按下键盘到字符出现在屏幕，中间都经历了哪些过程。

以Windows平台为例，当键盘上某个键被按下时，首先是操作系统进行处理，并产生相应的**按键事件**。随后这个事件被派发（Dispatch）到当前活跃的**窗体**（Window）。我们的Chrome浏览器作为Windows应用程序，也是通过Windows窗体来实现的，那么它也可以获得这个事件。这一点可以通过Spy++等软件确认。

![spy](chrome-spy.png)
<center><div style="font-size:16px;">Chrome Window in Spy++</div></center>

在Chrome UI上，各个组成部分其实也是一种窗体，包括搜索栏、地址栏、页面区域等。Chrome把这些窗体提供了一个抽象层`Auro`，同时Chrome维护了一个`DesktopWindowTreeHost`的窗体负责**派发来自操作系统的各类事件**。到达各个`Auro`窗体的事件，又会被发送到`View`层。
![dispatch](msg-dispatch.png)
<center><div style="font-size:16px;">Chrome Window Abstraction</div></center>

**View层**是Chrome设计的平台无关的UI框架，它将所有的View组织成一个**树结构**。你可以将View理解为浏览器**UI组件的一个抽象**，事件在各层View间逐层传递。比如，我们熟悉的Web页面就在`Web Contents`这一层View中渲染。

![view](view.png)
<center><div style="font-size:16px;">Views in Chrome</div></center>

在Auro Window抽象层中，消息和事件的传递需要**事件派发器**（Event Processor）和**事件定位器**（Event Targeter）共同完成。前者负责把后者找到的第一个事件，发送给目标对象处理。处理事件的可能是浏览器进程中的某个功能组件（`Widget`）。对于键盘事件，还可能会先调用`ui::InputMethod::DispatchKeyEvent`去处理，以便于**与系统输入法交互完成输入**。

一般情况下，调用相应组件的`NativeWidgetAura::OnEvent()`方法处理事件。而对于页面相关的事件，会调用`RenderWidgetHostViewAura::OnEvent()` 处理，这里就会把事件交给Chrome的**渲染引擎Blink**了。有些浏览器的**保留事件**，比如CTRL+T（打开新Tab页）的键盘事件，不会发送给Web页面处理。

![handling](msg-handling.png)
<center><div style="font-size:16px;">Chrome Event Handling</div></center>

事件到了View这一层，也有相应的事件定位器（Event Targeter），用于寻找处理事件的View实例。Web Contents相关的View拿到键盘事件，就可以交给渲染进程处理了。

前面的章节提到过，每个渲染进程（每个Tab都有一个）都维护了一个I/O线程，它会接受和发送其他进程（如浏览器进程、网络进程）的数据。来自**浏览器进程的键盘事件通过I/O线程到达渲染主线程**，并由主线程维护的`RenderViewHost::OnMessageReceived`处理。接着这个事件会被转换为标准的HTML事件对象进入DOM结构，开启我们熟悉的冒泡和捕获过程，处理函数依次被加入消息队列，等待被执行。

从上面的架构不难看出，Chrome的多层结构都有类似的特点，从Window、Widget到View和页面，在事件处理时的模型都是类似的。到了页面的DOM结构中，也依然按照树形进行分发和处理。

这就是一个键盘事件从被系统产生，到被浏览器中页面处理的完整流程，但只是提供了一个梗概，仅供参考。以上过程分析基于Chrome源码提供的相关文档<sup>[6][7]</sup>，如有理解不当之处请批评指正。

## 总结
在本文中，我们首先探讨了在处理键盘和输入法相关逻辑时，可能遇到的性能问题及其解决方案。然后，深入浏览器内部，探索了事件派发和处理的流程。在下一篇文章中，我会对这个系列做个总结。

## 参考阅读
- [1] [高性能JavaScript](https://book.douban.com/subject/5362856/)
- [2] [添加事件处理程序影响性能](https://stackoverflow.com/questions/28627606/does-adding-too-many-event-listeners-affect-performance)
- [3] [Event Delegation](https://javascript.info/event-delegation)
- [4] [Input Events from Browser's view](https://developer.chrome.com/blog/inside-browser-part4/)
- [5] [Improve Scrolling Performance](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener#improving_scrolling_performance_with_passive_listeners)
- [6] [Input Event in Chrome UI](https://chromium.googlesource.com/chromium/src/+/master/docs/ui/input_event/index.md#Background)
- [7] [Event Handling in Render](https://www.chromium.org/developers/design-documents/displaying-a-web-page-in-chrome/)