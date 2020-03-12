---
title: 日拱一卒系列（五）—— Angular 大列表性能优化
date: 2020-03-12 16:27:48
categories:
    - 技术
tags:
    - Tech
    - Web Front-end
    - Angular
    - 日拱一卒
---
![head](abstract.jpg)
### 引言

今天看到一篇文章，有关在Angular应用程序中，如何优化大列表的方法。当数据列表变得很大时，很多Web应用程序都会出现性能问题，作者认为这通常不是框架本身的问题，即框架是框架，代码不是好代码。很多呈现数据的代码没有很好地斟酌，导致了性能问题。下面来看具体的优化方案。

### 方法一：使用虚拟滚动条（Virtual Scrolling）

虚拟滚动条是[Angular CDK](https://material.angular.io/cdk/scrolling/overview)实现的一个组件，它可以用来比较高效地呈现数据量大的列表。虚拟滚动的原理是，**将列表的容器的高度设置为所有将要呈现数据项的总高度，但是只呈现在当前视图内的一部分数据**。我们从[这里](https://stackblitz.com/edit/cdk-virtual-scrolling-2?embed=1&file=src/app/app.component.ts)的例子可以看到，随着滚动条滚动，列表容器中呈现的列表项在不断变化，但是右侧的滚动条并不改变，只是改变了当前容器视图中的项目，并不是真的滚动。

```HTML
<cdk-virtual-scroll-viewport itemSize="10000">       
 <div *cdkVirtualFor="let item of items">
   {{ item }}
 </div>
</cdk-virtual-scroll-viewport>
```
<!--more-->

当然，虚拟滚动也有一些缺陷。比如，被隐藏的列表项并没有呈现，因此是不可搜索的，因为这里的“不可见”是指项目元素根本不在页面上，而不仅仅是CSS样式上的不可见。另一个缺陷是，这个虚拟滚动条的实现十分依赖于你项目的特定实现，你的组件实现如果很复杂，可能虚拟滚动条就不能正常工作了。另外，引入这个Angular CDK库，也会增加项目代码包的大小。

### 方法二：手动呈现

所谓“自动”呈现，是指用Angular提供的原生Directive `*ngFor` 来实现，而“手动”方式，则是直接调用Angular的API去呈现。比如用`ngFor`的实现如下：

```html
<tr *ngFor="let item of items; trackBy: trackById;" class="h-12">
  <td>
    <span>{{ item.label }}</span>
  </td>
</tr>
```

改成用Angular API的实现如下：

```html
<tbody>
  <ng-container #itemsContainer></ng-container>
</tbody>
<ng-template #item let-item="item">
  <tr class="h-12">
    <td>
      <span>{{ item.label }}</span>
    </td>
  </tr>
</ng-template>
```

列表的容器和模板定义如下。同时，我们可以利用 `ViewContainerRef`上的 `createEmbeddedView` 方法创建视图，针对每个列表项，创建一个模板项，然后作为容器的一部分呈现：
```typescript
@ViewChild('itemsContainer', { read: ViewContainerRef }) container: ViewContainerRef;
@ViewChild('item', { read: TemplateRef }) template: TemplateRef<any>;

//...
for (let n = start; n <= end; n++) {
    this.container.createEmbeddedView(this.template, {
      item: {
        label: Math.random()
      }
    });
}
//...
```
关于Angular容器和模板的用法可以参考官方文档，或者[这篇文章](https://blog.angular-university.io/angular-ng-template-ng-container-ngtemplateoutlet/)。采用上述实现后，对于10000规模的列表，脚本运行时间减少了近**1/4**。

### 方法三：逐步呈现（Progressive Rendering）

逐步呈现的思路是，先呈现列表的一个子集，推迟其他列表项的呈现。这需要用到一些事件循环（Event Loop）相关的函数。比如，我们可以利用 `setInterval` 函数，每10ms执行一次**手动呈现**，每次呈现100项，当所有项都呈现时，终结事件循环：

```typescript
//...
  let index = 0;
  const interval = setInterval(() => {
    const next = index + 100;
    for (let n = index; n <= next ; n++) {
      if (n >= length) {
        clearInterval(interval);
        break;
      }
      const item = {
        item: {
          label: Math.random()
        }
      };
      this.container.createEmbeddedView(this.template, item);
    }
    index += 100;
  }, 10);
//...
```
总结起来，后面两种方法是依赖项目特点的，并且是需要手动调用Angular API的，这可能引入一些bug，需要特殊处理。一般情况下，使用官方的Angular CDK库就可以满足很多需求了。

### 参考资料
* https://blog.bitsrc.io/3-ways-to-render-large-lists-in-angular-9f4dcb9b65
* https://blog.bitsrc.io/top-reasons-why-your-angular-app-is-slow-c36780a0a289
* https://material.angular.io/cdk/scrolling/overview
* https://blog.angular-university.io/angular-ng-template-ng-container-ngtemplateoutlet/
