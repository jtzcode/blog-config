---
title: 2019年 Web Component 入门教程（译）
date: 2019-07-10 20:46:51
tags:
    - Tech
    - Web Front-end
---
本文原载于： https://www.robinwieruch.de/web-components-tutorial/
作者：**Robin Wieruch**
原文标题：**Web Components Tutorial for Beginners [2019]**
- - - 

### 前言

_译者注：Web Component可以译为Web组件，不过也可以不翻译，从业者都能看懂，故后面译文都保留原词。_

在这个教程中你能够学到，如何构建第一个Web Component，以及如何将其运用在你的Web应用中。在我们开始前，先花点时间总体上了解下Web Component。近年来，Web Components，也称为自定义元素（Custom Elements），已经成为一些浏览器的标准API（参考[这里](https://caniuse.com/#feat=custom-elementsv1)），它让开发者能仅仅使用HTML，CSS，和JavaScript实现可重用的组件，而不需要依赖主流的前端框架如React，Angular和Vue。具体来说，自定义元素将页面的结构（HTML），样式（CSS）以及行为（JavaScript）封装为一个自定义HTML元素。例如，下面代码片段是一个HTML下拉列表组件的例子：

``` html
<my-dropdown
  label="Dropdown"
  option="option2"
  options='{ "option1": { "label": "Option 1" }, "option2": { "label": "Option 2" } }'
></my-dropdown>
```
在本教程中，我们将使用Web Component从头实现一个下拉列表组件。然后你可以在自己的应用或者其他地方重用这个组件，甚至可以在一些框架中使用，比如React（参考[这里](https://github.com/the-road-to-learn-react/use-custom-element)）。

### 为什么需要Web Components？

这里通过一个个人案例来解释Web Components的益处：不同职能的开发团队需要基于一个样式指南（Style Guide）创建一个UI库，但是有两个团队在实现组件时候采用了各自不同的框架：React和Angular。尽管两种实现都遵循样式指南，且组件的结构（HTML）和样式（CSS）上基本相同，但是组件的行为（比如，打开/收起下拉列表，选择列表的一项）是依据各自框架下的JavaScript的逻辑实现的。另外，一旦样式指南中有关于组件样式或行为的错误，每个团队会独立地修复这些问题，且并不一定能保证向后兼容。很快，两个团队的UI库就会在外观和行为上产生分歧。

<!--more-->

_作者注：这个错误与Web Components无关，而是一个样式指南中的常见错误。开发者可能在开发中不是积极地使用并更新该指南，而是仅仅作为文档参考一下，这样指南早晚会过期。_

最终，两个团队坐到一起讨论如何解决这个问题，并让我研究一下Web Component是否能解决。Web Component的确提供了一个令人信服的解决方案：两个团队使用标准的Web Components来实现需要组件，且仅仅使用HTML，CSS和JavaScript实现。这些组件可以在React和Angular应用中使用。如果样式指南发生改变，或者组件的而实现需要修复，两个团队可以协作完成对共享UI库的更新。

### 开始学习Web Components

如果你需要一个针对下面教程的初始项目，可以clone[这里](https://github.com/rwieruch/web-components-starter-kit)的GitHub代码。随着教程的进行，你可以在`/src`和`/dist`目录下做相应的改动。最终的项目代码可以在[这里](https://github.com/rwieruch/web-components-dropdown)找到。

现在我们开始构建第一个Web Component。我们不直接实现下拉列表，而是从一个简单的按钮控件开始，这个按钮最后会用到下拉列表上。使用Web Component实现一个简单的按钮听起来不太合理，因为你完全可以使用`<button>`元素配合一些CSS样式实现，使用按钮只是为了教学起见。下面的代码使用Web Component创建了一个按钮组件，包含了自定义的结构和样式。

``` javascript
const template = document.createElement('template');

template.innerHTML = `
  <style>
    .container {
      padding: 8px;
    }

    button {
      display: block;
      overflow: hidden;
      position: relative;
      padding: 0 16px;
      font-size: 16px;
      font-weight: bold;
      text-overflow: ellipsis;
      white-space: nowrap;
      cursor: pointer;
      outline: none;

      width: 100%;
      height: 40px;

      box-sizing: border-box;
      border: 1px solid #a1a1a1;
      background: #ffffff;
      box-shadow: 0 2px 4px 0 rgba(0,0,0, 0.05), 0 2px 8px 0 rgba(161,161,161, 0.4);
      color: #363636;
      cursor: pointer;
    }
  </style>

  <div class="container">
    <button>Label</button>
  </div>
`;

class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));
  }
}

window.customElements.define('my-button', Button);
```

我们一步一步来看。定义一个Web Component需要继承[HTMLElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement)这个[JavaScript类](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)。在派生的类中，你可以访问很多类方法，比如一些组件生命周期回调函数（Lifecycle Callbacks），这些方法能帮助你实现组件的功能。后面你会看到如何使用这些类方法。

另外，Web Components采用了[Shadow DOM](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM)机制，这有别于为了性能优化而采用的Virtual DOM技术。Shadow DOM将组件内部的HTML，CSS和JavaScript封装起来，使得这些内容对外部使用该组件的其他组件或者HTML不可见。你可以配置[Shadow DOM](https://developer.mozilla.org/en-US/docs/Web/API/Element/attachShadow)的模式（mode），比如这里将它的值设为`open`（ _译者注：原为是设置为true，根据文档来看有误_ ），意思是在某种程度上从外部可以访问到Shadow DOM的root节点。一句话，Shadow DOM 通过在你的自定义元素内部维护自己的DOM子树来实现结构和样式的封装。

在构造器中还有一条语句，在Shaodow DOM附加一个子节点，内容拷贝自前面已经声明的[模板](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template)。模板通常用来重用HTML代码，而在Web Component中模板的关键作用是，定义了组件所需的结构和样式（参考自定义元素前面的模板声明）。

上面代码段最后一行将我们的自定义元素作为一个有效元素定义在window对象上。第一个参数是自定元素的HTML标签名，该名字需要有一个横杠（`'-'`）；第二个参数就是自定义元素对象。今后，我们就可以在HTML中的某处使用这个自定义元素： `<my-button></my-button>` 。注意，自定义元素不能使用自关闭的标签（如`<my-button />`）。

### 如何向Web Components传递属性？

目前为止，我们的自定义元素除了拥有自己的结构和样式外没什么用处。我们完全可以使用原生的HTML元素结合CSS样式达到同样的目的。不过为了学习Web Component，我们还是继续完善这个自定义按钮元素。目前这个元素显示的内容是写死的，我们能不能尝试传递一个标签字符串作为它的HTML属性呢？比如：

``` html
<my-button label="Click Me"></my-button>
```

这样呈现的按钮的文本还是`Label`，为了按属性的值显示文本内容，我们可以监听（观察）这个属性的值，然后用它做一些事情，这些可以由继承自HTMLElement类的方法完成：

``` javascript
class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));
  }

  static get observedAttributes() {
    return ['label'];
  }

  attributeChangedCallback(name, oldVal, newVal) {
    this[name] = newVal;
  }
}
```

每次label属性的值改变，`attributeChangedCallback`方法都会被调用，因为我们在`observedAttributes`方法中将`label`属性定义为可观察属性。在这个例子里，我们的回调函数没有做太多事情，仅仅将`label`挂到Web Component实例上（效果相当于`this.label= 'Click Me'`）。然而，现在按钮还没有呈现这个`label`的值，为了达到目的你还需要抓取实际的HTML按钮元素并设置它的内容。 _（译者注：注意这里使用`shadowRoot`对象访问内部HTML元素）_ 

``` javascript
class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));

    this.$button = this._shadowRoot.querySelector('button');
  }

  static get observedAttributes() {
    return ['label'];
  }

  attributeChangedCallback(name, oldVal, newVal) {
    this[name] = newVal;

    this.render();
  }

  render() {
    this.$button.innerHTML = this.label;
  }
}
```

现在`label`属性的初始值被设置到按钮上，并且，自定义按钮还可以响应属性值的变化。你可以按照这种方式实现其他的属性。这里`label`属性是字符串类型，然而对于JavaScript中的非原生类型（non-primitives），比如`Object`和`Array`，你需要将其转换成JSON字符串后再传递给Web Components。稍后我们在实现下拉列表组件时会遇到这种情况。

### 将特性（Properties）反应到属性（Attributes）上

_（译者注：这里为了区别于属性，将properties翻译成”特性“ 。你可以简单理解：属性是HTML元素上的字段，特性是对象上通过get/set方法访问的字段。）_ 

到目前为止，我们使用属性将信息传递到自定义元素上。每当属性值发生改变，我们把改变后的值通过回调函数更新到Web Component实例的特性上，进而呈现出来。然而，我们也可以使用其他方法将属性值反映到特性上：JavaScript的[Get方法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes#Class_body_and_method_definitions)。这样，我们无需通过回调函数也能保证拿到最新的属性值，即`this.label`总是返回从`getter`函数拿到的最新的值。

``` javascript
class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));

    this.$button = this._shadowRoot.querySelector('button');
  }

  get label() {
    return this.getAttribute('label');
  }

  static get observedAttributes() {
    return ['label'];
  }

  attributeChangedCallback(name, oldVal, newVal) {
    this.render();
  }

  render() {
    this.$button.innerHTML = this.label;
  }
}
```

以上就是如何将属性值反应到特性上。然而，我们还可以通过特性（properties）将数据传递给自定义元素。当数据是类似JavaScript对象和数组时，这种方法很常用：

``` html
<my-button></my-button>

<script>
  const element = document.querySelector('my-button');
  element.label = 'Click Me';
</script>
```

不幸的是，我们通过特性更改数据时，之前的回调函数不会被调用，因为它们只会响应属性的改变。这时，我们需要一个`set`函数：

``` javascript
class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));

    this.$button = this._shadowRoot.querySelector('button');
  }

  get label() {
    return this.getAttribute('label');
  }

  set label(value) {
    this.setAttribute('label', value);
  }

  static get observedAttributes() {
    return ['label'];
  }

  attributeChangedCallback(name, oldVal, newVal) {
    this.render();
  }

  render() {
    this.$button.innerHTML = this.label;
  }
}
```
现在，由于我们在外部改变了元素的特性，自定义元素的`setter`方法会把这个改变反应到属性上，这样一来，回到函数又被调用了，自定义元素也可以重新呈现新的数据了。

你可以通过在每个方法中打log的方式来理解这些方法调用的顺序。你可以也可以通过浏览器的开发者工具查看DOM上属性的变化：新的属性值应该反应在元素上，即使它是通过特性改变的。

总的来说，有了`getter`和`setter`函数我们可以将信息以属性或者特性的方式传递给自定义元素。这个过程就称为将属性值反应到特性上，反过来也一样。

### 如何将函数传递给Web Components？

按钮最基本的功能是我们点击的时候有响应的动作。为了实现这个功能，首先，自定义元素可以注册一个事件监听器对用户的交互操作进行响应。如下面代码所示，我们可以获取button对象并附加一个事件监听器：

``` javascript
class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));

    this.$button = this._shadowRoot.querySelector('button');

    this.$button.addEventListener('click', () => {
      // do something
    });
  }

  get label() {
    return this.getAttribute('label');
  }

  set label(value) {
    this.setAttribute('label', value);
  }

  static get observedAttributes() {
    return ['label'];
  }

  attributeChangedCallback(name, oldVal, newVal) {
    this.render();
  }

  render() {
    this.$button.innerHTML = this.label;
  }
}
```

_作者注：你完全可以直接在自定义元素而不是内部的button上添加监听器。然而，在内部添加的好处是你可以对传入监听器的内容进行更多的控制。_

这里缺失的部分是外部的一个回调函数 _（译者注：即Web Component的用户希望事件发生时所做的处理）_ ，当事件响应时可以被自定义元素在内部被调用，这可以通过多种方法解决。首先，我们可以把这个函数以属性的方式传递。然而我们已经说过，将非原生类型当作属性传递比较笨重（需要转换成JSON字符串），因此我们要避免这么做。第二种方法就是将函数以特性的方式传递。代码如下：

``` html
<my-button label="Click Me"></my-button>

<script>
  document.querySelector('my-button').onClick = value =>
    console.log(value);
</script>
```

我们在自己的元素上声明了一个`OnClick`函数，后面会在自定义元素的监听器上调用这个函数：

``` javascript
class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));

    this.$button = this._shadowRoot.querySelector('button');

    this.$button.addEventListener('click', () => {
      this.onClick('Hello from within the Custom Element');
    });
  }

  ...

}
```

可以看到，你可以控制传递给外部回调函数的内容（参数），如果不在内部添加监听器，你就只能收到单击事件，而无法控制事件处理的一些细节，你可以试试看。虽然这个过程能正常工作，但我们希望只利用JavaScript DOM API提供的内置事件系统来实现。因此，我们不在自定义元素上定义特性，而是直接在外部注册一个事件监听器：

``` html
<my-button label="Click Me"></my-button>

<script>
  document
    .querySelector('my-button')
    .addEventListener('click', value => console.log(value));
</script>
```

输出跟前边做法一样，只是使用外部事件监听器处理`click`事件。这样，自定义元素依然可以使用click事件将信息发送到外部，因为自定义元素的内部实现依然发送了消息 _（译者注：这里`this.onClick`需要改成`this.onclick`调用，因为已经没有特性的定义，需改为内置的事件函数）_ ，最后在浏览器的控制台log中可以看见结果。

这里有一个限制是，在自定义元素上我们只能使用内置的事件。然而，如果你后面要在不同的环境（如React）上使用这个组件，你可能需要为你的组件提供自定义事件（比如`onClick` _（译者注：注意区别于内置事件`click`和处理函数`onclick`）_ ）API。当然了，你可以手动将内置的`click`事件的处理映射到自定义元素的`onClick`函数上，但是如果能使用内置的命名传统还是更方便的。我们来看看如何在前面基础上使用自定义事件达到目的。

``` javascript
class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));

    this.$button = this._shadowRoot.querySelector('button');

    this.$button.addEventListener('click', () => {
      this.dispatchEvent(
        new CustomEvent('onClick', {
          detail: 'Hello from within the Custom Element',
        })
      );
    });
  }

  ...

}
```

现在我们向外部暴露了一个自定义事件API，详细信息通过一个可选的`detail`特性传递。接下来，我们可以监听这个自定义事件：

``` html
<my-button label="Click Me"></my-button>

<script>
  document
    .querySelector('my-button')
    .addEventListener('onClick', value => console.log(value));
</script>
```

这最后一步使用自定义事件是可选的操作，只是提供了更多实现的可能性，如果它提供了你需要的，也许Web Component使用起来会更加轻松自如。

### Web Component生命周期回调

我们就要完成自定义按钮了。在我们用它实现自定义下拉列表前，还要完成最后一件事情。目前，这个按钮内部有个带有内边距（padding）的容器，这在多个按钮挨着排列时，彼此间形成一定的间距。然而，在其他的上下文，比如在下拉列表中使用该按钮时，你可能需要移除这个内边距。这时，你就可以使用Web Component的其中一个生命周期回调函数`connectedCallback`：

``` javascript
class Button extends HTMLElement {
  constructor() {
    super();

    this._shadowRoot = this.attachShadow({ mode: 'open' });
    this._shadowRoot.appendChild(template.content.cloneNode(true));

    this.$container = this._shadowRoot.querySelector('.container');
    this.$button = this._shadowRoot.querySelector('button');

    ...
  }

  connectedCallback() {
    if (this.hasAttribute('as-atom')) {
      this.$container.style.padding = '0px';
    }
  }

  ...

}
```

在我们的例子里，如果元素上有一个名为`as-atom`的属性，就将容器的内边距设为0。顺便说一下，这就是你在创建一个好的UI库时所遵循的“原子”设计原则。这里的按钮元素可以看作一个原子，而下拉列表可以看作一个分子 _（译者注：这是一种自底向上的软件设计方法，可参阅：https://speckyboy.com/atomic-web-design/）_ 。这些分子元素再往上层最后会形成“组织”。现在，我们的按钮在下拉列表中使用时可以不带内边距：`<my-button as-atom></my-button>`。按钮`label`属性稍后会设置。

但是这里`connectedCallback`生命周期回调什么时候调用呢？答案是每次Web Component被附加到DOM树上时被调用一次。还有一个等价的回调函数叫`disconnectedCallback`，它在每次Web Component从DOM树移除时调用。之前，你已经使用过`attributeChangedCallback`这个回调去响应属性值的变化了。Web Component提供的其他生命周期回调可以在 [这里](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements#Using_the_lifecycle_callbacks) 找到详细信息。

### Web Component的嵌套使用

