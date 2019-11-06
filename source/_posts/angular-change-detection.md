---
title: Angular 变更检测探究
date: 2019-11-05 10:26:37
tags:
    - Tech
    - Web Front-end
    - Angular
---

标题：<u>**Angular Service 变更检测探究**</u>

作者：**jtzcode**

上次更新：**2019年11月05日**

* * *
### 引言

使用过现代JavaScript框架的开发者，都应该熟悉绑定（binding）的概念。绑定通常有两个方向。一是由用户交互驱动，在浏览器的页面上发生了输入、点击操作，导致应用程序的状态发生改变，这些改变需要反映到程序中特定变量上。另一个方向是，JavaScript代码的业务逻辑中改变了程序的状态，比如通过API请求拿到新的数据，而这些状态也需要反映的页面的控件上。很多框架如AngularJS就实现了双向绑定机制。

下面我们来看上述第二种绑定，即程序状态从业务代码到前端页面的传递过程。如果沃恩自己去实现应该怎么做呢？最直观的想法是：在恰当的时候对程序中的变量表达式进行求值，看看它是否与原来的值相同。如果不同，则把新的值写入与页面渲染相关的对象中。这里面检查变量表达式的值是否变化的过程就是所谓的变更检测（change detection）。Angular就是采用变更检测实现绑定的。下面我们具体来看。

### 基本原理

程序状态变化的来源有以下几种，首先是响应用户在UI上的操作，比如用户点击了一个按钮，改变了某个变量的值。然后是浏览器的异步事件，比如`setTimeout`的回调函数，改变了某个变量。最后是应用程序中的异步事件，比如API返回的Promise或者Observable对象在Resolve时，改变了变量的值。那么Angular是如何知道这些状态的改变呢？

<!--more-->
我们知道，Angular的组件可以依赖其他的组件来构建应用程序的页面逻辑，最后形成一棵组件树。每个组件都有自己的变更检测器（change detector）。因此，变更检测器的结构也是一棵同构的树（见下图）。

![变更检测器树](change-detector.jpg)

当某个组件的状态发生改变时，Angular会从这棵树的根节点开始遍历，出发所有组件节点的变更检测器，这样Angular就知道那些组件的状态发生了改变，需要更新相应的UI（见下图）。这个过程看似开销很大，但Angular已经进行了大量优化，实际变更检测的速度很快。

![默认变更检测策略](default-detection.jpg)

上述变更检测的策略是Angular的默认行为。事实上，我们可以通过`ChangeDetectionStrategy`对象来配置某个组件的变更检测策略。如果不指定，该对象的值是`Default`。在默认情况下，某个组件的变更检测触发，受其他组件的影响。那么如何让组件只关注自己的内部的变化呢？答案是设置`ChangeDetectionStrategy`的值为`OnPush`。

### OnPush策略

在OnPush策略下，只有两种情况可以触发当前组件的变更检测：

* 组件的输入属性（绑定）的引用被改变
* 组件内部触发了异步事件

我们先来看第一点，这里关键词是**引用**。比如下面这个组件：
```typescript
@Component({
  template: `
    <test [config]="config"></test>
  `
})
export class AppComponent  {
  config = {
    name: 'jtz'
  };
  onClick() {
    this.config.name = 'code';
  }
}
```
我们在`AppComponent`中定义了`config`对象，并把它作为`<test>`组件的输入属性。`<test>`组件的变更检测策略定义为`OnPush`：
```typescript
@Component({
  selector: 'test',
  template: `
    <h1>{{config.name}}</h1>
    {{changeDetect}}
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class TestComponent  {
  @Input() config;
  get changeDetect() {
    console.log('Changed');
  }
}
```
在`AppComponent`中我们通过响应单击事件改变了`config`对象中字段的值，虽然config对象作为`TestComponent`的输入，但是实际执行就会发现，`changeDetect`方法并没有重新执行。原因就在于，我们在`AppComponent`中**改变的只是`config`对象`name`字段的值，而没有改变`config`本身的引用**。因此，如果想让变更检测触发，需要将`onClick`方法的实现改为：
```typescript
this.config = { name: 'code' };
```
即对`config`对象重新赋值，刷新引用。
下面来看在OnPush策略下触发变更检测的第二个因素：**组件内部的异步事件**。这里需要注意的是，这个异步事件特指页面的DOM事件，不包括定时器事件和AJAX请求返回等异步事件，对于后者Angular是无法直接获知事件的发生的。比如：
```typescript
@Component({
    template: `<span>{{count}}</span>`,
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class TestComponent {
    count = 0;
  
    constructor() {
      setTimeout(() => this.count = 1, 100);
      this.http.get('https://jtzcode.io').subscribe(data => {
        this.count = data.count;
      });
    }
  }
```
上面组件中，构造器中的异步事件触发后，更新了`count`的值，但这个更新不会反映到UI上，因为Angular并不知道异步事件的触发，因此没有触发变更检测。如果在组件UI上加一个按钮，单击按钮来更新`count`的值，就可以看到UI的改变。

OnPush策略除了可以减少不必要的变更检测从而提高性能，还可以方便开发者在特定的时刻触发变更检测，满足逻辑上的灵活性需求。比如，一个API请求返回一个Observable对象，我们需要利用该对象的数据流，在特定的时候更新组件的状态，这时我们可以通过Angular提供的接口来实现，比如：
```typescript
import { Component, Input, ChangeDetectionStrategy, ChangeDetectorRef, OnInit } from '@angular/core';
import { Observable } from 'rxjs';
@Component({
  selector: 'change-detect-observable',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
        <div>
            <div>Total items: {{counter}}</div>
            <button (click)="refresh()">Refresh</button>
        </div>
    `
})
export class ChangeDetectionComponent implements OnInit {
  title = 'change-detect-observable';
  @Input() items: Observable<number>;
  counter = 0;
  constructor(private changeDetector: ChangeDetectorRef) {
  }
  refresh() {
      console.log("Refresh counter.");
  }
  ngOnInit() {
    this.items.subscribe((v) => {
        this.counter++;
        if(this.counter % 2 == 0) {
            this.changeDetector.markForCheck();
        }
    }, null, () => {
        this.changeDetector.markForCheck();
    });
  }
}
```
在上面代码中，我们定义了一个名为`items`的Observable数据流对象，并在组件初始化时订阅了这个数据流。当数据流里产生了新的数据时，我们增加了计数器。注意，这里的`items`对象虽然是作为输入绑定，但在数据产生时，我们只是拿数据进行处理，对`items`对象本身没有操作，因此不会触发当前组件的变更检测，这符合我们的预期。

在这里，我们想在拿到的数字是偶数时，触发变更检测。为此，我们注入了一个类型为`ChangeDetectorRef`的Service，并调用了该Service的`markForCheck`方法。这个方法会手动触发当前组件的变更检测。从这个例子我们就能看到OnPush策略的灵活性。

### 生命周期钩子
有时候，我们需要监控绑定属性的变化，这时就可以使用组件的生命周期钩子（hook）函数`ngOnChanges`以及`ngDoCheck`。

OnChanges钩子会在组件的一个或多个绑定属性被改变后调用，我们可以在对应的`ngOnChanges`方法中拿到变化的属性，以及该属性的原值和新值，进而做相应处理。使用该钩子函数，需要让组件实现`OnChanges`接口。

DoCheck钩子函数会更细致一些，它可以利用Angular提供的差分器（differ）判断某个属性的改变是什么类型的，以进行相应操作。比如当属性是一个列表时，差分器可以判断该列表的变化是新增了一项，还是删除了一项。使用该钩子，需要让组件实现`DoCheck`接口。注意，如果你同时实现了DoCheck和OnChanges接口，那么DoCheck的实现会覆盖OnChanges的实现。

我们可以更深入一些来看看这些钩子函数是如何被调用的。先看下面这张图：

![变更检测过程](change-detection.png)

上图说明的是，在父组件中更新子组件的绑定属性时，Angular做的事情。还以前面介绍OnPush策略时的代码为例，父组件为AppComponent，子组件为TestComponent，在子组件有一个输入属性为`config`，该属性的值在父组件中由`config`字段传入。

当父组件的`config`发生改变并触发变更检测时，首先父组件更新子组件的绑定，即子组件中的`config`值。然后执行**子组件**的一系列生命周期钩子函数，比如ngOnInit，ngOnChanges等。注意，这一步还没有执行**子组件**的变更检测。接下来是父组件的DOM树重新渲染。渲染结束后，才执行组件的变更检测，这时就会发现子组件`config`属性的变化，且这个变更检测过程与父组件类似，是个**递归**过程。最后还会执行一些子组件的其他生命周期函数，如AfterViewInit等。这里要注意的是，子组件的重新渲染是在它自己的变更检测流程中进行的，图中没有显示，这说明OnChange等生命周期在渲染前就执行了。

以上就是关于Angular变更检测的介绍，欢迎讨论。

* * *
### 参考资料
* Angular权威教程
* https://blog.angularindepth.com/everything-you-need-to-know-about-change-detection-in-angular-8006c51d206f
* https://netbasal.com/a-comprehensive-guide-to-angular-onpush-change-detection-strategy-5bac493074a4