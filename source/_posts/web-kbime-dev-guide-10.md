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
首先考虑JavaScript本身的写法。通常来说，我们给一个输入控件添加事件处理程序（如keydown）后，这个处理程序会在短时间内频繁被调用，比如输入一段文字。如果是与鼠标拖动和滚动相关的事件，可能会更加频繁。此时，在处理程序内部的JavaScript的性能问题就值得关注了。尽管目前浏览器的渲染引擎和JS引擎性能都比较强大，对于复杂业务逻辑来说，注重代码的性能仍是十分有益的。


### 事件处理器数量
最好不要给过多的元素添加事件处理器。[实验表明](https://web.archive.org/web/20170121035049/http://jsperf.com/click-perf)，这种做法会显著降低页面性能，尽管每个元素只绑定一个事件处理程序[2]。这还只是绑定事件的操作，没有涉及具体的事件处理程序被调用的性能。你可以从这个基于jQuery的实验页面找到一些数据：

![perf](perf.png)
<center><div style="font-size:16px;">添加事件处理程序的性能</div></center>

通过`.many`样式添加的`click`事件处理程序会作用于大量元素，因此速度最慢。而通过父元素添加事件处理程序有着最好的性能，这就是**事件委托**（Event Delegation）。

### 事件委托
事件委托就是为了解决上述问题，通过在父元素添加事件处理程序，避免给过多子元素添加。理想情况下，我们只需要给**document文档对象**添加需要的事件处理器即可：
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

因为事件处理程序需要在主线程中执行，因此合成线程需要在合适的时候通知主线程调用事件处理程序。合成线程会把页面上**有事件处理程序的区域**标记为“**非快速滚动区域**”（Non-Fast Scrollable Region）。如果不在这类区域中，合成线程会直接合成下一帧，而不会等待主线程，从而优化性能。

![render](nonfast-scroll.png)
<center><div style="font-size:16px;">非快速滚动区域</div></center>

在事件委托的场景下，可能整个页面都在监听连续事件，那么整个document都是非快速滚动区域，那么合成线程就会不停地**等待主线程的执行**结果。不过此时可以给事件处理程序传递`passive`属性进行优化，详细可以参考文档[5]。

## 深入事件处理
下面我们以Chrome为例深入浏览器内部探究一下，从用户按下键盘到字符出现在屏幕，中间都经历了哪些过程。

以Windows平台为例，当键盘上某个键被按下时，首先是操作系统进行处理，并产生相应的**按键事件**。随后这个事件被派发（Dispatch）到当前活跃的**窗体**（Window）。我们的Chrome浏览器作为Windows应用程序，也是通过Windows窗体来实现的，那么它也可以获得这个事件。这一点可以通过Spy++等软件确认。

![spy](chrome-spy.png)
<center><div style="font-size:16px;">Chrome Window in Spy++</div></center>

以上过程分析基于Chrome源码提供的相关文档[6]，如有理解不当之处请批评指正。

## 总结
## 参考阅读
- [1] 高性能JavaScript
- [2] [添加事件处理程序影响性能](https://stackoverflow.com/questions/28627606/does-adding-too-many-event-listeners-affect-performance)
- [3] [Event Delegation](https://javascript.info/event-delegation)
- [4] [Input Events from Browser's view](https://developer.chrome.com/blog/inside-browser-part4/)
- [5] [Improve Scrolling Performance](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener#improving_scrolling_performance_with_passive_listeners)
- [6] [Input Event in Chrome UI](https://chromium.googlesource.com/chromium/src/+/master/docs/ui/input_event/index.md#Background)