---
title: Groovy有何不同
date: 2020-11-10 10:41:59
categories:
    - 技术
tags:
    - Tech
    - Groovy
    - Java
---
## 引言
## 动态类型
Groovy是动态强类型语言
语言类型动态与静态、强与弱的区别。
Groovy的动态类型：能力式设计vs契约式设计、多态的实现、多方法、可选类型、静态类型检查、编译、性能损失。
## 闭包
对闭包的支持，让程序更加灵活，也不想Java中匿名函数那么复杂。
除了语法上的优雅，闭包还为函数将部分实现逻辑委托出去提供了一种简单、方便的方式。
闭包能够扩充、优化或增强另一段代码。闭包有两个非常擅长的具体领域：一个是辅助资源清理，另一个是辅助创建内部的领域特定语言（DSL）。

针对闭包的用法可以给出一个简单、典型的例子（此处有代码）

闭包实际上实现了策略模式。（此处可以展开策略模式的定义和案例）
闭包对资源的使用流程尤其有用，可实现为`Execute Around Method`模式。除了用于处理资源释放，还可以捕获异常、上传必要数据、打印日志等。

闭包支持协程。协程（`Coroutine`）则支持多个入口点，每个入口点都是上次挂起调用的位置。我们可以进入一个函数，执行部分代码，挂起，再回到调用者的上下文或作用域内执行一些代码。之后我们可以在挂起的地方恢复该函数的执行。本质上是控制流在不同的方法（或闭包）之间来回转移，共同完成一项任务（比如生产者-消费者模式）。
（此处可以有代码）

闭包科里化：当对一个闭包调用`curry()`时，就是要求预先绑定某些形参。在预先绑定了一个形参之后，调用闭包时就不必重复为这个形参提供实参了。这在**多次调用一个闭包而部分参数总是相同**时十分有用。
（此处应该有代码）

Groovy的闭包支持方法委托，而且提供了方法分派能力，这点和JavaScript对原型继承（prototypal inheritance）的支持很像。在闭包内引用的变量和方法都会绑定到`this`，它负责处理任何方法调用，以及对任何属性或变量的访问。如果`this`无法处理，则转向`owner`，最后是`delegate`。

利用闭包优化递归性能：
```groovy
def factorial
factorial = { int number, BigInteger theFactorial ->
    number == 1 ? theFactorial : factorial.trampoline(number - 1, number * theFactorial)
}.trampoline()

println "factorial of 5 is ${factorial(5, 1)}"
println "Number of bits in the result is ${factorial(1000, 1).bitCount()}"
```
当我们调用`trampoline()`方法时，该闭包会直接返回一个特殊类`TrampolineClosure`的一个实例。当我们向该实例传递参数时，比如像`factorial(5, 1)`中这样，其实是调用了该实例的`call()`方法。该方法使用了一个简单的`for`循环来调用闭包上的`call`方法，直到不再产生`TrampolineClosure`的实例。这种简单的技术在背后将递归调用转换成了一个简单的迭代。

## 字符串
