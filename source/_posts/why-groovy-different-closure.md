---
title: Groovy有何不同 —— 闭包篇
date: 2020-11-25 14:53:00
categories:
    - 技术
tags:
    - Tech
    - Groovy
    - Java
    - 闭包
    - 编程语言
---
## 闭包
与很多动态类型语言类似，Groovy也有对闭包的支持。这既能让程序更加灵活，也不像Java中匿名函数那么复杂。除了语法上的优雅，闭包还为函数将部分实现逻辑委托出去提供了一种简单、方便的方式。闭包可以作为函数的参数和返回值，这使得Groovy中也支持**高阶函数**。

![groovy](groovy-logo.png)

### 常用场景
闭包能够扩充、优化或增强另一段代码，你可以理解为**闭包在给函数打补丁**。闭包有两个非常擅长的具体领域：一个是辅助资源清理，另一个是辅助创建内部的领域特定语言（DSL）。比如：
<!--more-->
```groovy
def static consume(closure) {  
    def r = new FileResource()  
    try {    
        r.open()    
        closure(r)  
    } 
    finally {   
        r.close()  
    }
}

consume() { r->
    def text = r.read()
    println(text)
}
```
`consume`方法接受一个闭包对象为参数，这个闭包内部是使用资源进行工作的核心代码，而资源的打开和清理工作就交给了consume函数，使用者不需要关心资源的清理工作。除了用于处理资源释放，还可以捕获异常、上传必要数据、打印日志等。

这实际上实现了**策略模式**，每个闭包都是一种策略。比如你要实现一个运算函数，这个函数可以处理加法、减法和乘法，这时你可以把操作的部分代理给闭包去做，只需要传入操作数即可。函数的用户需要什么操作只需要传入自定义的闭包即可。

### 协程和科里化
闭包还支持协程。协程（`Coroutine`）支持多个入口点，每个入口点都是上次挂起调用的位置。我们可以进入一个函数，执行部分代码，挂起，再回到调用者的上下文或作用域内执行一些代码。之后我们可以在挂起的地方恢复该函数的执行。本质上是控制流在不同的方法（或闭包）之间来回转移，共同完成一项任务（比如生产者-消费者模式）。例如：

```groovy
def loop(n, closure) {  
    1.upto(n) {    
        closure(it)  
    }
}
loop(5) {  
    total += it  
}
```
`loop`函数接收两个参数，第一个是循环的次数，第二个是循环的具体任务，这是一个闭包。在这里闭包完成对当前循环次数的累加，我们可以看到，累加过程不是一气呵成的。程序的控制流在`loop`函数和闭包之间来回切换。

Groovy闭包还支持科里化：当对一个闭包调用`curry()`时，就是要求预先绑定某些形参。在预先绑定了一个形参之后，调用闭包时就不必重复为这个形参提供实参了。这在**多次调用一个闭包而部分参数总是相同**时十分有用。例如：

```groovy
def testCurry(closure, date){
    printCurry = closure.curry(date)  
    printCurry "Curry"  
    printCurry "Stephen"
}
testCurry() { date, name ->  
    println "Date is ${date}. Name is ${name}"
}
```
本来传入`testCurry`方法的闭包需要两个参数，但使用`curry`函数后，相当于把`date`参数固定下来，接下来的调用只需要传递一个参数`name`。

Groovy的闭包还支持方法委托，而且提供了方法分派能力，这点和JavaScript对原型继承（prototypal inheritance）的支持很像。在闭包内引用的变量和方法都会绑定到`this`，它负责处理任何方法调用，以及对任何属性或变量的访问。如果`this`无法处理，则转向`owner`，最后`delegate`。更多信息可参考资料2。

### 递归优化
在Groovy中，可以利用闭包优化递归性能。比如计算较大整数的阶乘，朴素的递归程序可能消耗大量的内存。Groovy提供了`trampoline`方法进行优化：

```groovy
def factorial
factorial = { int number, BigInteger theFactorial ->
    number == 1 ? theFactorial : factorial.trampoline(number - 1, number * theFactorial)
}.trampoline()

println "factorial of 5 is ${factorial(5, 1)}"
println "Number of bits in the result is ${factorial(1000, 1).bitCount()}"
```
当我们调用`trampoline()`方法时，该闭包会直接返回一个特殊类`TrampolineClosure`的一个实例。当我们向该实例传递参数时，比如像`factorial(5, 1)`中这样，其实是调用了该实例的`call()`方法。该方法使用了一个简单的`for`循环来调用闭包上的`call`方法，直到不再产生`TrampolineClosure`的实例。这种简单的技术在背后将递归调用**转换成了一个简单的迭代**。

## 参考资料
1. Groovy程序设计 [美] Venkat Subramaniam
2. https://groovy-lang.org/closures.html#_delegation_strategy