---
title: 日拱一卒系列（一）—— Angular启动过程简析
date: 2020-01-10 16:13:15
tags:
    - Tech
    - Web Front-end
    - Angular
    - 日拱一卒
---
![head](abstract.jpg)
## 引言
这篇文章简要介绍一下Angular应用程序的启动（**Bootstrap**）原理。这里的内容来源于网上的一些关于Angular的学习资料，只会涉及基本的原理和过程，不会涉及过于细节的源代码，感兴趣的话大家可以自己研究源代码。

目前的Angular的大版本是8，即将发布Angular 9，这篇文章是基于Angular 8的。

我们在开始学习Angular并动手做一些练习时，可能会好奇，一个简单的 `ng serve`命令背后发生了什么，让Angular程序能够开始运行，并在浏览器中呈现预期的页面。为什么自己写的Angular Component和Module等代码块会变成最终的、能被浏览器识别的代码。这些就是本文关注的问题。

## 原理

我们知道，Angular框架为了简化开发，提供了Module、Component以及Decorator等编程组件。由于这些特性都不是JavaScript原生的功能，浏览器是不认识的，因此程序跑起来前需要一次编译过程。这个过程就是由**Angular Compiler**完成，对应的工具就是`ngc`。至于说这个编译器做了什么事情，就跟Angular最终呈现DOM对象的机制有关。

### DOM 模型
你可能听说过在React框架中采用了所谓的**虚拟DOM（Virtual DOM）** 的模型。它的基本原理是，当页面有元素的改动时，就会创建一个新的DOM对象，然后计算新旧DOM对象之间的差异。依据计算结果生成一个转换命令序列（Transformation Series），执行这个序列去更新DOM中需要改动的元素，这样就避免了整个DOM的刷新。
<!--more-->

Angular没有采用这种方式，而是采用了一种称为**增量DOM（Incremental DOM）** 的模型。在这种模型下，每个组件都被编译成一系列JavaScript DOM API指令，包括**创建元素**的指令，以及**更新元素**的指令。当变更检测机制发现页面需要更新，只需要找到正确的指令集合然后执行，就可以实现页面的部分更新。我们可以注意到，这里面有个组件的编译过程，这个就是Angular Compiler的工作。

如果想看看ngc工具可以编译出哪些文件，可以先写一个简单的Hello World程序，然后在`package.json`里的`scripts`部分添加一行命令：
```JavaScript
"compile": "ngc"
```
最后运行`npm run compile`即可。在dist目录下可以找到编译生成的文件，如下图：

![ngc](ngc-result.png)

这些文件在Angular应用程序启动时，被用来实例化各种对象，以及页面元素的呈现。


### 启动过程

在浏览器中，Angular基本在index页面执行，首先会启动一个叫Angular运行时（Angular Runtime）的对象。该对象会先确定当前运行应用的浏览器平台是什么，因为这会决定后面生成的DOM API指令序列，不同浏览器对API的支持有差异。这个工作由`
@angular/platform-browser-dynamic`模块实现。平台确定后，就会通过`bootstrapModule`方法启动入口模块，比如AppModule：

```JavaScript
platformBrowserDynamic().bootstrapModule(AppModule)
    .catch(err => console.log(err));
```
接下来，入口模块会被实例化，包括其依赖的的所有可注入的provider。`
@NgModule`装饰器会告诉Angular运行时如何实例化一个Module，以及如何注入需要的Service。在编译过程中产生的Module工厂函数（Module Factory Function，可以参考`app.module.ngfactory.js`文件）完成Module的实例化，并启动入口组件（Component），比如`app-root`。例如：

```javascript
var AppModuleNgFactory = i0.ɵcmf(i1.AppModule, [i2.AppComponent], function(_l) {...}


/**
 * Bootstrap all the components of the module
 */
    _moduleDoBootstrap(moduleRef) {
      ...
      const appRef = moduleRef.injector.get(ApplicationRef) as ApplicationRef;
      ...
      // loop over the array defined in the @NgModule, bootstrap:              [AppComponent]
      moduleRef._bootstrapComponents.forEach((
        // *** Follow bootstrap() ***
        // bootstrap the root component *AppComponent* with selector *app-root*
        f => appRef.bootstrap(f)));
      ));
    }
```
Module对象会遍历入口组件的数组，并对于每个入口组件调用应用程序对象的`bootstrap`方法。到此为止的过程是，**Platform**对象创建**Module**对象，Module对象得到**Application**对象，然后使用Application对象，启动每个Module上定义的入口**Component**。

入口组件的启动过程，就是将编译时生成的DOM API序列在浏览器中执行。首先，按组件的定义（HTML、CSS）创建出需要的页面元素，然后在需要的时候运行更新的指令，改变页面元素。这个过程目前是由**Angular View Engine**完成的，在未来的版本中，会被**Angular Ivy**取代。后者的一个优势是，可以对编译结果进行**tree-shaking**，即“摇晃掉”不需要的代码。

至此，Angular应用程序就启动完成了，页面也会呈现出来。本文旨在提供Angular启动过程的简单梳理，如果对具体细节感兴趣，可以详细阅读参考资料的文章，以及Angular源代码和编译后的临时代码。

## 参考资料


* https://blackat.github.io/angular/how-angular-bootstrap/?utm_campaign=Angular%20Weekly&utm_medium=email&utm_source=Revue%20newsletter
* https://blackat.github.io/angular/angular-ivy-introduction/
* https://blog.nrwl.io/understanding-angular-ivy-incremental-dom-and-virtual-dom-243be844bf36
