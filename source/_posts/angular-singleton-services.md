---
title: Angular Service 单例问题探究
date: 2019-10-15 10:26:37
tags:
    - Tech
    - Web Front-end
    - Angular
---

标题：<u>**Angular Service 单例问题探究**</u>

作者：**jtzcode**

上次更新：**2019年10月12日**

* * *

### 引言 

Angular Service是一种实现代码抽象的方式。它可以帮助我们将业务逻辑从页面的呈现逻辑中分离出来，以保持Component功能的纯净，以及某些常见功能的重用。一些开发者认为Component应该只处理页面呈现以及用户交互方面的逻辑，其他的都应该抽象到service中，即所谓的[Lean Angular Component](https://blog.angularindepth.com/lean-angular-components-252bcb6ea6c1)思想。


我们在使用Angular Service时会有这种场景：有一个变量的状态是全局有效的，在不同的Component中都可能修改这个状态值，并且每个Component中都可以拿到这个Service中该状态变量的最新值。为了实现这个场景，一个最关键的点就是，Service必须是单例的。因为一旦有多个实例，那么每个Component看到的状态变量的值就不一致了。这篇文章就来试图探讨一下，什么情况下Angular Service会产生多个实例，以及如何解决这个问题。

Angular的官方文档关于[单例Service](https://angular.io/guide/singleton-services)的话题有不错的讨论。除了这篇文章，本文还参考了其他讨论类似话题的文章，再结合了自己的理解。欢迎批评和讨论。

### 什么时候会有多个实例？

总的来说在两种情况下，Angular会生成多实例的Service：

* 声明Service的共享Module被多个其他Module导入

* 声明Service的共享Module被其他lazy-load的Module导入

这两种情况看起来比较类似，但其实背后的原理有些不同，我们分别来看。
<!--more-->
第一种情况是很常见的做法：我们定义了一个Service，比如是关于获取用户数据的，自然就会把它归到跟用户逻辑相关的Feature Module中，比如Customer Module：

```typescript
@NgModule({
    providers: [CustomerService],
    declarations: [CustomerComponent],
    entryComponents: [CustomerComponent]
})
export class CustomerModule {}
```
这个`CustomerService`在应用的其他模块也可能需要，其他模块的Component在注入前，需要导入`CustomerModule`模块。这个用法在Angular Router的应用场景中很常见。当我们需要分离Router的定义时，就要定义不同Routing Module（比如CustomerRoutingModule），并导入到各自逻辑对应的Module中，而这些Routing Module都需要导入Angular的Router模块。然而，Router模块中有Router Service，这就满足上述第一种情况，**Router模块被多个模块导入，导致Router Service产生多个实例。** 因此，Angular自身也需要解决这个问题。
在说明Angular如何解决这个问题之前，我们先来看一个细节。Angular的根模块（通常是AppModule）在编译时，会沿着模块的树状结构收集，沿路上模块所声明的所有Provider，并将其合并到同一个数组中，统一提供一个Injector用于依赖注入。也就是说，虽然Angular的模块依赖关系是一棵树，但最终会被合并成一个模块，且Injector的结构是扁平化的。想了解更多这方面的内容，可以参考[这里](https://blog.angularindepth.com/avoiding-common-confusions-with-modules-in-angular-ada070e6891f)。
然而，虽然Injector是同一个，但采用在Provider数组中声明Service并导入模块的方式使用Service，仍然是多实例的。就上面的例子来说，在某个Module中使用依赖注入时，每次遇到类型为`CustomerModule`的模块导入，就调用统一的Injector实例化一个`CustomerService`的对象。

### 如何解决？

Angular提供了三种方法解决这个问题：

* 使用`providedIn`语法替代在模块中注册Service

* 将Service实现在调用Module中

* 在Module中定义`forRoot()`和`forChild()`静态方法

我们一一来看。

*<u>使用`providedIn`语法</u>*
这种语法是**Angular 6.0**之后开始支持的，[官方文档](https://angular.io/guide/singleton-services)也有介绍。providedIn语法表示在应用的根模块上提供这个Service，这样我们不用在任何Module的provide数组中再声明这个Service。例如：

```typescript
import { Injectable } from '@angular/core'; 
@Injectable({ 
    providedIn: 'root'
})
export class UserService {
}
```
*<u>将Service实现在调用Module中</u>*
这相当于是每个Module实现了自己的Service，当然不会有多实例的问题。但当程序规模变大时，很多Service难免需要跨Module共享，这种方法就失效了。

<u>*在Module中定义forRoot()和forChild()静态方法*</u>
这个方法你可能比较眼熟，对了，就是在使用Angular的路由模块时，用于路由的注册。例如：
```typescript
const appRoutes: Routes = [
  { path: 'test', component: TestComponent },
  { path: '', redirectTo: '/test', pathMatch: 'full' },
  { path: '**', component: PageNotFoundComponent }
];

@NgModule({
  imports: [
    RouterModule.forRoot(
      appRoutes,
      { enableTracing: true } 
    )
  ],
  exports: [
    RouterModule
  ]
})
export class AppRoutingModule {}
```
现在你可以理解了为什么路由模块要定义一个`forRoot`方法：Angular的RoutingModule会经常被及其他模块引用，而这个Module里又定义了一些必要的Service，为了保证这些Service是单例的，RoutingModule实现了`forRoot`方法。

那么为什么`forRoot`或者`forChild`方法会起作用呢？事实上，这两个方法名只是推荐的习惯用法，你在自己Module里定义的方法不一定叫这个名字，只要满足一定的规范，就可以实现让Module提供单例的Service。我们来看一个`forRoot`方法的实现：
```typescript
static forRoot(config: UserServiceConfig): ModuleWithProviders {
  return {
    ngModule: TestModule,
    providers: [
      {provide: UserServiceConfig, useValue: config }
    ]
  };
}
```
我们可以看到`forRoot`是一个静态方法，返回 [ModuleWithProviders](https://angular.io/api/core/ModuleWithProviders) 接口类型。这个类型需要一个`ngModule`字段和一个`providers`数组字段。这个方法可以理解为Angular提供的另外一种声明Module的方法，通过`forRoot`这个静态方法拿到Module的实例。

这样如果仅仅在`AppModule`里面调用一个共享Module的`forRoot`方法一次，那么这个共享Module提供的Service就是全局有效的，其他导入Module的地方都使用`forChild`，就不会产生新的实例。

关于`forRoot`和`forChild`的区别，奥妙就在于**实现它们时，是否在`providers`中指定Service**。`forRoot`方法就会指定，`forChild`就不会，这也是为什么前者只能调用一次，而后者可以调用多次的原因。如果`forRoot`方法被多次调用，依然可能产生多实例的Service。

当然你也可以在两个方法中提供两组不同的Service，当需要导入某一组时，就在导入Module时调用相应的方法。

`forRoot`方法的使用如下：
```typescript
@NgModule({
  imports: [
    TestModule.forRoot({userName: 'JTZ CODE'}),
  ],
})
```
这样就会保证`TestModule`中声明的Service是单例的。

在实际应用中还有一种例外情况是，Module是通过路由延迟加载的（lazy-load）。延迟加载的Module的特点是它有自己的Injector去实例化Service。前面我们提到，Angular在编译时会将所有Module合并成一个Module，但是延迟加载的Module是动态的，它们不会被合并。因此，如果你在一个延迟加载的Module中引用了一个共享Module，那么其中的Service的实例也是只在这个延迟加载的Module中有效，这就可能产生多实例的Service。

因此，对于有延迟加载模块的场景，一个好的习惯是不要在共享的Module中声明Service，而是将Service注册到较高层次的Module上，或者使用`providedIn`语法。

关于Service单例的问题，Angular官网文档的讨论已经比较全面了，如果要深入了解Service依赖注入以及单例的控制机制还需要参考源代码。

* * *
### 参考资料

* https://blog.angularindepth.com/avoiding-common-confusions-with-modules-in-angular-ada070e6891f

* https://angular.io/guide/router#refactor-the-routing-configuration-into-a-routing-module

* https://angular.io/guide/singleton-services

* https://medium.com/@cyrilletuzi/understanding-angular-modules-ngmodule-and-their-scopes-81e4ed6f7407