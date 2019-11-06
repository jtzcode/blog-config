---
title: 什么是高阶函数？你为什么需要关注它？（译）
date: 2019-09-25 14:58:43
tags:
    - Tech
    - Web Front-end
    - JavaScript
---
**本文原载于**： [https://jrsinclair.com/articles/2019/what-is-a-higher-order-function-and-why-should-anyone-care/](https://jrsinclair.com/articles/2019/what-is-a-higher-order-function-and-why-should-anyone-care/)
**作者**：James Sinclare
**原文标题**: What Are Higher-Order Functions, and Why Should You Care?

*James Sinclare写于2019年7月2日*

* * *

**高阶函数**（high-order function）是一个被人们反复提及却经常被误用的概念，但是人们从未放弃过解释它的含义。也许你已经知道什么是高阶函数了，但我们如何在现实场景中使用它呢？有没有一些实际的例子能说明如何以及何时该使用高阶函数？我们能使用它操作DOM吗？或者，那些使用高阶函数的人只是在炫耀？他们有没有为此给代码引入不必要的复杂性？

我认为高阶函数是很有用的。事实上，我觉得它是JavaScript这门语言最重要的特性之一。但我们在解释它为何重要之前，先来看看什么是高阶函数。为了解释这个概念，我们从函数变量（functions as variables）开始。

### 作为一等公民的函数

在JavaScript中，我们至少有三种方式定义一个函数（注释1）。首先我们可以写一个函数声明（function declaration），例如：
```javascript
// Take a DOM element and wrap it in a list item element.
function itemise(el) {
    const li = document.createElement('li');
    li.appendChild(el);
    return li;
}
```
我希望你已经熟悉这种写法了，但你很可能还知道函数表达式（function expression）的写法，就像这样：
```javascript
const itemise = function(el) {
    const li = document.createElement('li');
    li.appendChild(el);
    return li;
}
```
还有另外一种写法就是箭头函数（arrow function）了：
```javascript
const itemise = (el) => {
    const li = document.createElement('li');
    li.appendChild(el);
    return li;
}
```
对我们来说，这三种写法本质上是相同的（注释2）。但是请注意，在后两个例子中函数被赋值给了变量，看起来是件小事，为什么不这么做呢？而事实上，这至关重要。
<!--more-->

JavaScript中的函数是”一等的“，这是因为我们可以：

* 将函数赋值给变量

* 将函数作为其他函数的参数传递

* 将函数作为其他函数的返回值（注释3）

看起来不错，不过这与高阶函数有什么关系呢？注意上面列出的最后两点，我们一会儿回来着重来看，同时举几个例子。
我们已经看过将函数赋值给变量了，那将函数作为参数呢？我们来写一个可以操作DOM元素的函数。如果我们运行和函数`document.querySelectorAll()`，就会得到一个`NodeList`。注意这个列表不是`array.NodeList`，它不像数组那样有`.map()`方法。因此，我们自己实现一个：
```javascript
// Apply a given function to every item in a NodeList and return an array.
function elListMap(transform, list) {
    // list might be a NodeList, which doesn't have .map(), so we convert
    // it to an array.
    return [...list].map(transform);
}
// Grab all the spans on the page with the class 'for-listing'.
const mySpans = document.querySelectorAll('span.for-listing');

// Wrap each one inside an <li> element. We re-use the
// itemise() function from earlier.
const wrappedList = elListMap(itemise, mySpans);
```
在这个例子中，我们把`itemise`函数作为参数传递给`elListMap`函数。然而，`elListMap`函数不仅可以创建列表。例如，我们也可以使用它给一系列元素添加样式：
```javascript
function addSpinnerClass(el) {
    el.classList.add('spinner');
    return el;
}

// Find all the buttons with class 'loader'
const loadButtons = document.querySelectorAll('button.loader');

// Add the spinner class to all the buttons we found.
elListMap(addSpinnerClass, loadButtons);
```
我们的`elListMap`函数接收一个`transform`函数作为参数。这意味着我们可以重用它完成各种不同的任务。现在我们已经看到如何将函数作为参数传递，那如何将函数作为返回值呢？
我们从一个标准的函数开始，想要将一个 `<li>`元素列表包裹进一个`<ul>`元素中。这不是很难：
```javascript
function wrapWithUl(children) {
    const ul = document.createElement('ul');
    return [...children].reduce((listEl, child) => {
        listEl.appendChild(child);
        return listEl;
    }, ul);
}
```
那如果我们一会儿又想把一些段落元素包裹进一个`div`元素呢？没有问题，我们可以写一个类似的函数：
```javascript
function wrapWithDiv(children) {
    const div = document.createElement('div');
    return [...children].reduce((divEl, child) => {
        divEl.appendChild(child);
        return divEl;
    }, div);
}
```
上面的函数可以工作，但是这两个函数太过类似了，唯一的不同是我们创建的父亲元素（parent element）的类型。当然了，我们可以再传入一个参数表示父元素的类型，但这里有另外一种做法。我们可以构造一个函数，它的返回值是另外一个函数，就像这样：
```javascript
function createListWrapperFunction(elementType) {
    // Straight away, we return a function.
    return function wrap(children) {
      // Inside our wrap function, we can 'see' the elementType parameter.
      const parent = document.createElement(elementType);
      return [...children].reduce((parentEl, child) => {
          parentEl.appendChild(child);
          return parentEl;
      }, parent);
    }
}
```
现在这段代码乍一看有点复杂，我们分解来看。我们创建了一个函数，做一些事情，最后返回另一个函数。但是，这个返回的函数记住了`elementType`参数。这样，稍后我们调用这个返回函数时，它就知道需要创建什么类型的元素。因此，我们可以这样来实现`wrapWithUl`和`wrapWithDiv`：
```javascript
const wrapWithUl  = createListWrapperFunction('ul');
// Our wrapWithUl() function now 'remembers' that it creates a ul element.

const wrapWithDiv = createListWreapperFunction('div');
// Our wrapWithDiv() function now 'remembers' that it creates a div element.
```
上面提到的返回函数”记住“一些数据的这种做法有一个术语叫**闭包**（注释4）。闭包使用起来就不那么方便了，不过我们现在不用为此担心。综上，我们已经看到：

* 将函数赋值给变量

* 将函数作为其他函数的参数传递

* 将函数作为其他函数的返回值

总的来说，使用上述的一等函数看起来不错，但是这与高阶函数有什么关系呢？下面我们来看高阶函数的定义。


### 什么是高阶函数？

高阶函数是：

>这样一类函数，它接收一个函数作为参数，并返回一个函数作为结果。（注释5）

听起来没什么新鲜的？在JavaScript中函数是一等公民，高阶函数就是表达这个意思。这没什么大意思，只是把一个简单的术语包装的高大上（fancy-sounding）一些。

**高阶函数的例子**

一旦你开始往下看，你会发现很多高阶函数。其中最常见的是接受函数作为参数的高阶函数，我们先来看这一种。然后，我们再看返回值是其他函数的高阶函数例子。

<u>接受函数作为参数的函数</u>

每当传递了一个”回调“函数，你就是在使用高阶函数了，这种用法在前端开发中到处可见。最常见的要数`.addEventListener()`方法了，我们经常使用该方法对各类事件作出响应动作。例如，如果我要点击按钮弹出提示框：
```javascript
function showAlert() {
  alert('Fallacies do not cease to be fallacies because they become fashions');
}

document.body.innerHTML += `<button type="button" class="js-alertbtn">Show alert</button>`;

const btn = document.querySelector('.js-alertbtn');

btn.addEventListener('click', showAlert);
```
在个例子中，我们首先创建了一个函数用于弹出提示框，然后在页面上添加一个按钮，最后把`showAlert()`函数作为参数传递给`btn.addEventListener`。
还有一类常见的高阶函数是[数组遍历方法](https://jrsinclair.com/articles/2017/javascript-without-loops/)，就像`.map()`，`.filter()`和`.reduce()`之类的方法。我们已经在`elListMap`方法中见识过了：
```javascript
function elListMap(transform, list) {
    return [...list].map(transform);
}
```
高阶函数还可以帮助我们处理延时和计时相关的代码，常用的函数是`setTimeout`和`setInterval`。例如，我们想在30秒后移除一个高亮的样式，可以这样做：
```javascript
function removeHighlights() {
    const highlightedElements = document.querySelectorAll('.highlighted');
    elListMap(el => el.classList.remove('highlighted'), highlightedElements);
}

setTimeout(removeHighlights, 30000);
```

我们再次创建了一个函数并把它作为参数传递给另一个函数。可以看到，这种做法在JavaScript中很常见，并且你可能已经这样使用过了。

<u>返回值是函数的函数</u>

相比函数作为参数传递，函数作为返回值不是那么常见，但它仍然很有用。最有用的一个例子是`maybe`函数，我基于 [Reginald Braithewaite的实现]( https://leanpub.com/javascriptallongesix/read) 修改了一个版本如下：
```javascript
function maybe(fn)
    return function _maybe(...args) {
        // Note that the == is deliberate.
        if ((args.length === 0) || args.some(a => (a == null)) {
            return undefined;
        }
        return fn.apply(this, args);
    }
}
```

在分析这个函数的实现之前，我们先来看看如何使用它。回顾之前的`elListMap`函数：
```javascript
// Apply a given function to every item in a NodeList and return an array.
function elListMap(transform, list) {
    // list might be a NodeList, which doesn't have .map(), so we convert
    // it to an array.
    return [...list].map(transform);
}
```

如果我们不小心传递了一个`null`或者`undefined`值给`elListMap`函数会发生什么？我们会得到一个`TypeError`异常，所有做的事情都会终止。而`maybe`函数就可以解决这个问题。我们这样使用：
```javascript
const safeElListMap = maybe(elListMap);
safeElListMap(x => x, null);
// ￩ undefined
```

这个函数会返回`undefined`而不是终止执行，而如果我们继续把返回值传递给另一个由`maybe`保护的函数，则会再一次返回`undefined`。我们可以使用`maybe`函数保护任意多的这样的函数调用，比我们写一堆if语句来判断好得多。
返回值是函数的函数在React社区中很常见。比如，react-redux中的`connect`函数就是一个例子。

**那又怎么样？**

我们已经看到了一些独立的高阶函数的例子，可这些有什么用呢？没有它们对我们有什么影响吗？有没有比这些特意构造的例子看起来更有用的东西？为了回答这个问题，我们再来看一个例子。考虑JavaScript内置的`sort`函数，它的问题在于改变了已有的数组，而不是返回一个新数组（*译者注：这不符合函数式编程的规范*）。这个暂且不提，`sort`函数本身就是一个高阶函数，它的其中一个参数是函数。这个函数如何使用呢？如果我们要对一个数组排序，首先需要定义一个比较函数，例如：
```javascript
function compareNumbers(a, b) {
    if (a === b) return 0;
    if (a > b)   return 1;
    /* else */   return -1;
}
```

然后，可以这样对数组排序：
```javascript
let nums = [7, 3, 1, 5, 8, 9, 6, 4, 2];
nums.sort(compareNumbers);
console.log(nums);
// 〕[1, 2, 3, 4, 5, 6, 7, 8, 9]
```

这是对数字的排序。但是这很有用吗？我们多大程度上需要对一个数字的数组排序？恐怕不是很需要。我们需要排序的往往是对象数组。例如：
```javascript
let typeaheadMatches = [
    {
        keyword: 'bogey',
        weight: 0.25,
        matchedChars: ['bog'],
    },
    {
        keyword: 'bog',
        weight: 0.5,
        matchedChars: ['bog'],
    },
    {
        keyword: 'boggle',
        weight: 0.3,
        matchedChars: ['bog'],
    },
    {
        keyword: 'bogey',
        weight: 0.25,
        matchedChars: ['bog'],
    },
    {
        keyword: 'toboggan',
        weight: 0.15,
        matchedChars: ['bog'],
    },
    {
        keyword: 'bag',
        weight: 0.1,
        matchedChars: ['b', 'g'],
    }
];
```

试想我们把这个对象数组按照字段`weight`来排序。我们本可以写一个新的排序方法，但这没必要，我们只需要写一个新的比较函数就行了：
```javascript
function compareTypeaheadResult(word1, word2) {
    return -1 * compareNumbers(word1.weight, word2.weight);
}

typeaheadMatches.sort(compareTypeaheadResult);
console.log(typeaheadMatches);
// 〕[{keyword: "bog", weight: 0.5, matchedChars: ["bog"]}, … ]
```

我们可以写一个针对任意数组排序的比较方法。`sort`函数与我们达成协议说：”如果你给我一个比较函数，我帮你对任何数组排序。别担心数组里面有什么，给我比较函数，我就帮你排序。”所以我们不必纠结于排序算法的实现，只需要专注于两个元素的比较的简单任务即可。

试想如果没有高阶函数，我们就不能把一个函数传递给`sort`方法，那么一旦有新的数组需要排序，我们就需要写一个新的排序方法。或者，我们选择重新发明类似函数和对象指针的东西来模拟高阶函数。不管哪种方式都是很笨拙的。

有了高阶函数，我们可以将比较函数从排序函数中分离出来。试想一个聪明的浏览器工程师想出一个更快的排序算法，并更新了`sort`函数的实现，那所有人的代码都会受益，不管你排序的数组里是什么内容。事实上，[有一整套数组的高阶函数](https://jrsinclair.com/javascript-array-methods-cheat-sheet)遵循这个模式。

这让我们想的更深入一些。`sort`方法把排序任务从需要排序的数组内容抽象出来，这是一种关注点分离（separation of concerns）。高阶函数提供了创建抽象的方式，以避免写出笨拙的代码。软件工程领域80%的事情都是关于创建抽象的。

每当我们移除重复代码进行重构时，就是在创建抽象。我们发现了一种模式，并用该模式的一种抽象表示来替代它，这会让我们的代码更精简且易于理解。至少，这就是更深入的想法。

高阶函数是一种创建抽象的强大工具。跟抽象相关的有一整套数学理论：**范畴论**（Category Theory）。更精确地说，范畴论所关注的是寻找抽象的抽象。或者说，是寻找模式背后的模式（pattern）。在过去70多年中，聪明的程序员一直在借鉴这些思想，将其应用到编程语言的特性和代码库中。如果我们学习这些模式背后的模式，有时就可以减少很多不必要的代码，或者将复杂问题简化为若干简单构造块（building block）的优雅组合，而这些构造块就是高阶函数。这就是高阶函数很重要的原因：我们有了另一种对抗代码复杂性的有力工具。

* * *

如果你想了解更多高阶函数的知识，可以参考：

* 《高阶函数》（ https://eloquentjavascript.net/05_higher_order.html）：《Eloquent JavaScript》第五章，作者 Marijn Haverbeke

* 《高阶函数》（ https://medium.com/javascript-scene/higher-order-functions-composing-software-5365cf2cbe99）：Eric Elliott写的《 Composing Sofware》系列文章的一部分

* 《JavaScript高阶函数》（ https://www.sitepoint.com/higher-order-functions-javascript/）：M. David Green在Sitepoint撰写的文章

你可能已经在使用高阶函数了。JavaScript让使用过程非常简单以至于我们都感觉不到在使用它。当听到别人讨论这个概念时，知道他们在说什么总是有用的。高阶函数的概念本身并不复杂，但是简单概念背后是一个强大的工具。

* * *

**2019年7月3日更新**：如果你有函数式编程的经验，就会发现我使用了非纯函数（Impure Functions）以及一些冗长的函数名。这不意味着我不了解非纯函数和一般的函数式编程原则，或者我在产品代码里使用这样的函数名。这只是一个教程的代码片段，因此我选用了一些初学者更容易理解的实际例子，有时候就会做出妥协。如果你感兴趣，可以参看我在其他地方写的关于[函数纯度](https://jrsinclair.com/articles/2018/how-to-deal-with-dirty-side-effects-in-your-pure-functional-javascript/)（functional purity）以及[函数式编程的一般原则](https://jrsinclair.com/articles/2016/gentle-introduction-to-functional-javascript-intro/) 的文章。

* * *

### 注释
1、定义一个函数至少有三种方式，我们另找时间讨论这个话题。
2、这不总是正确的。三种定义函数的方式在实践中都有微妙的区别。这种区别在于神奇的this关键字的上下文以及trace栈中的标签。
3、 维基百科贡献者（2019）， 'First–class citizen',Wikipedia, the free encyclopedia, viewed 19 June 2019, https://en.wikipedia.org/wiki/First-class_citizen.
4、如果你想了解更多关于闭包的知识，可以参考写的《 Master the JavaScript Interview: What is a Closure? 》
5、 高阶函数（2014）, viewed 19 June 2019, http://wiki.c2.com/?HigherOrderFunction. 





