---
title: Groovy有何不同 —— 
date: 2020-11-30 09:51:39
tags:
---
## 字符串
Groovy会把使用单引号创建的String看作一个纯粹的字面常量。因此，如果在里面放了任何表达式，Groovy并不会计算它们；相反，它就是按所提供的字面内容来使用它们。要对String中的表达式进行求值运算，则必须使用双引号。这里有一个GString类型的陷阱（逻辑依赖类型来定义）。
Groovy支持字符串惰性求值，即使用变量拼接的字符串，在使用后改变变量的值，字符串也会相应更新。
Groovy支持多行字符串。
Groovy支持更加丰富的字符串操作，这得益于对Java运算符的重载和实现。比如`-=`操作符对于操纵字符串很有用，它会将左侧的字符串中与右侧字符串相匹配的部分去掉。Groovy在`String`类上添加的`minus()`方法使其成为可能。
为方便匹配正则表达式，Groovy提供了一对操作符：`=~`和`==~`，`=~`执行RegEx部分匹配，而`==~`执行RegEx精确匹配。

## 集合类型
Groovy在`Collection`接口上添加的方法：`each()`、`collect()`、`find()`、`findAll()`、`any()`和`every()`。


## GDK小技巧
Groovy中`with()`方法是作为`identity()`的同义词引入的，所以它们可以互换使用。该方法接受一个闭包作为参数。在闭包内调用的任何方法都会被自动解析到上下文对象。
```groovy
lst = [1, 2]
lst.with {  
    add(3)  
    add(4)  
    println size()  
    println contains(2)
}
```