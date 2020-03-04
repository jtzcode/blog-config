---
title: 日拱一卒系列（二）—— 理解 TypeScript 类型
date: 2020-03-02 14:06:27
categories:
    - 技术
tags:
    - Tech
    - Web Front-end
    - TypeScript
    - 日拱一卒
---
![head](abstract.jpg)
## 引言
这篇文章谈谈有关 TypeScript 类型的一些概念和原理。学习资料来源于网络，欢迎讨论。感兴趣的朋友可以阅读参考资料中的原始文章。

## 如何理解 TypeScript 类型

首先，我们可以将TypeScript的类型理解为**一组值（Values）的集合**。如果变量a属于A类型，那么所有可以赋值给变量a的元素就组成了A类型的集合。例如：

```typescript
let a: A;  /* a is of type A */
```

如果A类型的变量可以赋值给B类型的变量，那么A类型所有可能的实例值也是B类型可能值的一部分。例如：

```typescript
let a: A; /* A type can be assigned to B type*/
let b: B = a;
```

如果是由类型A，B和C的联合（Union）定义的类型ABC，它的可能值的集合就是A，B和C三种类型所有可能值得集合。例如：

```typescript
type ABC = A | B | C;  /* ABC is union of type A, B, C */
```
<!--more-->
我们还可以将类型理解为**一组类型兼容的关系（Type Compatibility Relationships）**。TypeScript定义了一些类型关系，可以在[这里](https://github.com/microsoft/TypeScript/blob/master/doc/spec.md#3.11)找到，文档中定义了诸如子类型、超类型以及赋值兼容性等，也就是说一种类型在什么情况下可以赋值给另一种类型。例如，A类型要想赋值给B类型，需要满足以下某个条件（完整列表见上述文档）：
* A 和 B 是完全相同的类型
* A 或者 B 类型是 `any`
* A 是 `Undefined` 类型
* 其他条件...

另外，文档中的 `Apparent Members` 部分就定义了联合类型的兼容性。TypeScript的一个特点是，一个变量在不同的位置可以有不同的静态类型，例如：
```typescript
const array: any[] = [];
array.push(1234);
// inferred type: number[]
array.push('abcd');
// inferred type: (string | number)[]
```

## 名义（Nominal）类型系统和结构化（Structural）类型系统

一般来讲，编程语言的类型系统要么属于名义类型系统，要么就是结构化类型系统。二者的区别如下：
* 在名义类型系统中，如果说两个类型相等，那么两个类型的名字（identity）一定相同；如果一个类型是另一个类型的子类型，则必须显式声明。常见的采用名义类型系统的编程语言有C++，Java和C#等。
* 在结构化类型系统中，如果说两个类型相等，只要它们有相同的结构定义即可（每个字段名字相同，且类型相等）。如果A类型是B类型的子类型，只需要B类型包含所有A类型的字段结构即可。**TypeScript 就采用了结构化的类型系统**。

```typescript
class A {
  title = 'A';
}
class B {
  title = 'B';
}
const ab: A = new B(); 
// A and B are equal types

class C {
  title = 'C';
}
class D {
  title = 'D';
  ID = 123;
}
const c: C = new D();
// C is a sub-type of D
const d: D = new C();
// Error: missing 'ID' in type C but required in type D
```
在上面例子中，A类型与B类型由于具有相同的结构，即是相同的类型。类型D包含了类型C的结构，就可以用来实例化C类型的对象，反过来就会报错。

## 参考资料

* https://2ality.com/2020/02/understanding-types-typescript.html
* https://github.com/microsoft/TypeScript/blob/master/doc/spec.md#3.11
* https://www.typescriptlang.org/docs/handbook/type-compatibility.html