---
title: Jenkins Pipeline 手记（1）—— 什么是CPS编程
date: 2020-09-25 15:43:56
categories:
    - 技术
tags:
    - Tech
    - Jenkins
    - CPS
    - DevOps
    - 日拱一卒
---
### 引言
最近在工作中使用Jenkins进行持续集成任务，遇到这样一个问题：
```shell
java.io.NotSerializableException: java.util.regex.Matcher
```
经过调查发现这是Jenkins的CPS插件报出的，原因是在使用正则表达式相关操作的函数时，代码写法不规范，导致不能序列化。那么什么是CPS呢， Jenkins为什么又需要做序列化的操作呢？

### 什么是CPS

在函数式编程中，**CPS (Continuation-Passing Style**）是一种编程风格：**所有的控制块都通过 continuation 来显式传递**。简单来说，在CPS风格中，函数不能有返回语句，它的调用者要想获得它的结果，需要显式传递一个回调函数来获取结果并继续执行。而为了保证整个程序执行下去，这个回调函数还会一直嵌套下去。这里的回调函数就是一个“continuation”。暂时找不到好的翻译，就保留它的原文。
<!--more-->
下面来看一个递归的例子，比如常见的计算数学阶乘的函数：
```javascript
// 原来代码
function fact(n) {
  if (n == 0)
    return 1 ;
  else
    return n * fact(n-1) ;
}

// CPS风格代码
function fact(n,ret) {
  if (n == 0)
    ret(1) ;
  else
    fact(n-1, function (t0) {
     ret(n * t0) }) ;
}

// 调用入口
fact (5, function (n) {
  console.log(n) ; // Output 120
})

```
上面是传统的递归实现，下面是CPS风格的实现。可以看到return语句被替换成一个回调函数的调用，而相乘这个动作会在接下来的递归里通过嵌套的回调函数完成，直到终止条件。你可以理解为，在每一层的递归中，回调函数都会给参数多乘上一个系数，并调用上一层的回调函数，直到最外层打印结果。

### CPS的应用场景
**1. 在Jenkins中的应用**
回到最初的问题，Jenkins为什么会使用CPS编程技术呢？原因是在Jenkins Pipeline的实现中，期望在任何时候都可以中断代码的执行，保存状态，并在适当时候恢复执行。这可以应对Jenkins agent宕机的场景。如果一个函数执行过后就返回了，那么就会丢失一部分状态，CPS代码由于在中间不返回结果，因此可以解决这个问题。

然而，你也的Pipeline代码是Groovy语言的，其本身并不是CPS风格的，这就需要一个解释器将代码编译成CPS风格，Jenkins里面通过 [groovy-cps](https://github.com/cloudbees/groovy-cps/) 这个插件来完成。你也可以查看这个插件的稳定获得更多的关于CPS和Groovy解释器的介绍。

我们知道，如果想要持久化一个对象的状态，那么这个对象需要时可序列化的（Serializable）。Jenkins如果想要保存Pipeline的状态，就会要求CPS的代码也是可序列化的。如果有一些代码是不可序列化，或者序列化不安全的，我们可以在Pipeline代码中加上`@NonCPS`标记。并且，声明了该标记的函数也只能调用其他有该标记的函数。Jenkins会对这类函数进行常规编译，不生成可序列化的CPS代码。

在写跟正则表达式相关的功能时，可能会用到的`~`操作符就是序列化不安全，因此要声明标记`@NonCPS`：
```groovy
def t = (text =~ TEST_REGEX)
```
如果忘记在使用上面语句的方法前声明该标记，就会抛出文章开头的不可序列化的异常，我遇到的问题的原因就在此。


**2. 在异步请求中应用**
另外一个我们容易想到的场景就是异步请求，我们在JavaScript代码里经常用回调函数来处理请求的结果，并保持程序不被挂起。这其实就一种CPS风格：

```javascript
request("./index", function (text) {
 document.getElementById("test").innerHTML = text ;
}) ;
```
request函数的实现使用Ajax对象：
```javascript
function request (url, onSuccess, onFail) {
 
  var req ;
  function process() {
    if (req.readyState == 4) {
      if (req.status == 200) {
        if (onSuccess)
          onSuccess(req.responseText, url, req) ;
      } else {
        if (onFail)
          onFail(url, req) ;
      }
    }
  }
  if (window.XMLHttpRequest)
    req = new XMLHttpRequest();
  req.onreadystatechange = process;
  req.open("GET", url, async);
  req.send(null);
 
  return req ;
}

```
这里面的回调函数`onSuccess`和`onFail`就是所谓的 **continuation**。

### 参考资料

* http://matt.might.net/articles/by-example-continuation-passing-style/
* https://github.com/jenkinsci/workflow-cps-plugin
* https://github.com/cloudbees/groovy-cps/
* https://en.wikipedia.org/wiki/Continuation-passing_style