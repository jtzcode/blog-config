---
title: 日拱一卒系列（四）—— React 组件重用探究
date: 2020-03-05 15:43:21
categories:
    - 技术
tags:
    - Tech
    - Web Front-end
    - React
    - 日拱一卒
---
![head](abstract.jpg)
## 引言
今天看了两篇关于React组件重用的文章，作者给出了一些小技巧，方便我们更好地考虑组件重用的设计。现在分享在这里。

## 重用的思路

我们通常意义理解的可重用，一般是指能适配多种用户常见，可以被其他用户使用，满足他们的需求。但是基于这种理解实现的可重用组件会有一个问题：**忙于扩展功能以适应新需求，导致组件的实现越来越复杂，bug也可能越来越多**。

例如，如果想适配更多的用户需求，我们最直接的做法是增加属性的数量，即定义新的props元素：
```jsx
<ReusableComponent name={this.state.name} description={description} />
```
<!--more-->
当我们要加一个新的属性`status`时，就需要改写props数组，添加处理逻辑，然后在使用时传入新的属性。

```jsx
<ReusableComponent name={this.state.name} description={description} status="initialized" />
```
随着时间推移，这个属性列表会越来越长，组件内部的处理逻辑会越来越复杂。我们可以想想为什么会出现这种情况。一方面我们有重用并扩展组件的需求，而同时我们也需要每个属性有明确的功能。为每个新功能增加一个属性，牺牲了设计的简洁性，但功能一目了然。我们不太可能同时要求可扩展、设计简单又易用。因此，我们可以牺牲一些易用性，来提高扩展的方便性，比如：
```jsx
<ReusableComponent values={this.state.values} handlers={handlers} />
```
我们可以把组件的属性值和处理函数抽象出来，分别放到一个数组中，这样既满足了扩展的需求，又不至于增加太多的属性，但是牺牲了可读性和易用性。组件的使用者不能通过属性名就得知该组件的用法。同时，组件的内部实现将更加抽象和复杂，但这是将开发的难度转移到组件设计者而不是让使用者承担。

我想在真正实现时，还是要做一个权衡：哪些属性是需要直接暴露出来的，哪些可以直接集成在一起，很多组件的设计可能类似这样：

```jsx
<ReusableComponent name={this.state.title} onChange={(e) => this.setState()} otherOptions={this.state.options} />
```
这个组件直接暴露了 `name` 和 `onChange` 属性，将其他属性隐藏在 `otherOptions` 中。

## 复杂一点的例子

下面来看一个稍微复杂一些的例子，在某个组件内部还有子组件。如果这时要扩展父组件，还要考虑新增的属性对子组件的影响。例如：

```jsx
<TicketList
    price={prices}
    isRoundtrip
    items={labels}
/>
```
这是一个展示机票列表的组件，它的实现类似：

```jsx
...
{condition && <TicketListItem label={items[0]} {..otherProps}/>}
...
```
注意，这里的 `TicketListItem` 并不是公开的，而是作为组件 `TicketList` 实现的一部分，外部程序只能使用 `TicketList` 组件。这种设计就不利用重用，因为 `TicketList` 的实现依赖于它的子组件，而外部使用者无法直接干预子组件达到修改父组件的目的。更好的做法是将父组件和子组件都设计为公开的接口，这样用户可以灵活使用和组合需要的组件，并且修改其中一个组件不会影响其他组件的使用。

```jsx
// expose both components to end users

import {TicketList, TicketListItem} from 'shared-ui'

// let them compose these themselves
<TicketList>
   <TicketListItem 
		label="Trip Fare" 
		price={600}
		{...others} />
   <TicketListItem 
		label="Seat Upgrade" 
		price={450} />
</TicketList>
```
## 参考资料
* https://www.ohansemmanuel.com/thinking-in-react-isnt-enough/?utm_campaign=React%2BNewsletter&utm_medium=web&utm_source=React_Newsletter_200
* https://www.ohansemmanuel.com/props-overload-why-more-props-doesnt-mean-more-reusable/